---
title: "ClickHouse 완전 정복 10장 — 삽입 패턴: 작은 삽입의 함정과 async insert"
date: 2026-06-13 23:49:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "운영"]
---
3장에서 MergeTree의 핵심 성질 하나를 못박았다.
**part는 불변이고, 한 번의 삽입은 part 하나를 만든다.**
그때는 "그래서 삽입이 빠르다"는 좋은 면만 봤다.
이 장은 그 동전의 뒷면이다 — **잘못 삽입하면 ClickHouse가 무릎을 꿇는다.**

ClickHouse 운영에서 신입이 가장 먼저, 가장 자주 만나는 에러가 하나 있다.

```
DB::Exception: Too many parts (3000).
Merges are processing significantly slower than inserts.
```

이 한 줄에 이 장의 모든 것이 들어 있다.
왜 이 에러가 나는지, 그리고 이걸 피하는 두 갈래 길 — **큰 배치**와 **async insert** — 을 본다.
이건 운영 레벨 이야기라, 개념보다 "실제로 어떻게 넣어야 안 터지나"가 중심이다.

## 먼저, 왜 삽입 방식이 문제가 되나

3장의 사실 두 개를 다시 깐다(공식 문서 기준).

- **각 INSERT 문은 영향받는 파티션마다 최소 1개의 새 part를 만든다.**
- part는 불변이라, 늘어난 part는 **백그라운드 머지**가 모아서 큰 part로 통합한다.

여기서 산수가 어긋나면 문제가 생긴다.

| 삽입 방식 | 만들어지는 part |
|---|---|
| 1만 행을 **1번**에 삽입 | part **1개** |
| 1만 행을 **1만 번** 개별 삽입 | part **1만 개** |

같은 데이터인데 part 수가 1만 배 차이 난다.
삽입이 여러 파티션에 걸치면 part는 파티션 수만큼 더 늘어난다.

머지는 part를 점진적으로 합치는 **백그라운드 작업**이고, 속도에 한계가 있다.
작은 part가 머지가 따라잡을 수 있는 속도보다 빨리 쌓이면, part가 무한정 늘어나기 시작한다.
공식 문서가 진단을 정확히 짚는다 — 근본 원인은 **"머지가 삽입보다 현저히 느리게 처리되고 있다"**는 것.

> 비유: 머지는 책상 위에 쌓이는 서류를 **한 권의 바인더로 정리하는 사서**다.
> 누가 서류를 한 번에 한 장씩, 1초에 수백 장씩 던지면 사서는 정리를 포기한다.
> 책상은 서류로 뒤덮이고(part 폭발), 결국 ClickHouse가 "더는 못 받는다"고 선언한다.
> 이건 이해를 돕는 비유다 — 실제 임계값과 동작은 바로 아래에서 숫자로 본다.

## 'Too Many Parts'의 실제 임계값

ClickHouse는 part가 너무 쌓이면 두 단계로 방어한다(소스 기준, 26.x).

| 설정 | 기본값 | 단일 파티션 active part가 이를 넘으면 |
|---|---|---|
| `parts_to_delay_insert` | 1000 | 삽입을 **인위적으로 지연**시킴(브레이크) |
| `parts_to_throw_insert` | 3000 | 삽입을 **예외와 함께 거부**(Too Many Parts) |

검사는 MergeTree 코드의 `delayInsertOrThrowIfNeeded` 함수가 한다.
즉 part가 1000개를 넘으면 "천천히 좀 넣어라"고 삽입을 늦추고, 3000개를 넘으면 아예 막아버린다.
이건 단일 **파티션**당 active part 기준이다.

> 참고로 일부 블로그·구버전 문서는 임계를 '300 parts'로 인용한다.
> 이는 23.6 이전 구버전 기준이거나 설명용으로 단순화한 수치로 보이고, 현재(26.x) 소스 기본값은 1000/3000이다.
> 정확한 버전 경계는 공개 출처 간 차이가 있어 단정하지 않는다.
> 어느 쪽이든 교훈은 같다 — **작은 part를 쌓지 마라.**

## 작은/잦은 삽입이 비싼 진짜 이유

단순히 part 수가 많아지는 것만 문제가 아니다.
작은 삽입이 부담스러운 이유는 여러 겹이다(공식 블로그 기준).

- **part마다 파일이 생긴다** — Wide 포맷이면 컬럼당 파일까지.
  part가 많으면 파일 수가 폭증한다.
- **write amplification** — 작은 쓰기가 잦으면 CPU·IO가 실제 데이터양에 비해 과하게 든다.
- **지속적 머지 비용** — 머지는 part를 합칠 때 **압축 해제 → 정렬 → 재압축**을 한다.
  작은 part가 많을수록 이 작업이 끝없이 돈다.
- **복제 환경에선 더 비싸다** — 8장의 ReplicatedMergeTree에선 part 하나하나가 ClickHouse Keeper에 엔트리를 남긴다.
  작은 part가 많으면 Keeper 부담까지 커진다.

그래서 ClickHouse의 삽입 철학은 한 줄로 요약된다.
**"적게, 그러나 크게 넣어라."**

## 길 1 — 큰 배치 삽입 (가능하면 이게 최선)

가장 단순하고 가장 좋은 답은 **클라이언트가 모아서 한 번에 크게 넣는 것**이다.
공식 문서의 권장 수치다(공식 문서 기준).

| 항목 | 권장값 |
|---|---|
| 최소 배치 | insert당 **1,000행 이상** |
| 최적 배치 | **1만~10만 행**(넓게는 10만~50만 행) |
| 삽입 빈도 | **1~2초당 1회** (초당 여러 번 X) |
| 단일 part 용량 | 기본적으로 약 100만 행까지 |

핵심은 빈도다.
초당 100만 행을 처리해야 하더라도, 그걸 **잘게 100만 번**이 아니라 **1~2초마다 한 번의 큰 INSERT**로 묶어 보내면 안전하다.

```sql
-- 1~2초당 1회, 1만~50만 행을 한 번에
INSERT INTO events
SELECT * FROM input('ts DateTime, user_id UInt64, payload String')
FORMAT Native;  -- 클라이언트가 10K~100K행 모아 1개 INSERT로 전송
-- → 파티션당 part 1개만 생성, 머지 부담 최소화
```

데이터가 어쩔 수 없이 **행 단위**로 들어온다면(예: 이벤트 스트림), 클라이언트 측에 버퍼를 두고 일정량/일정 시간마다 모아서 보낸다.
반대로 한 번에 너무 거대한 삽입이라면 적당히 쪼갠다(throttle).
공개적으로 확인되는 범위에선, 단일 노드 처리량이 대략 초당 50만 행을 넘어가면 클라이언트 배칭만으론 한계라 클러스터 서버 증설(수평 확장)을 고려하라고 본다.

문제는 — **클라이언트가 항상 배칭할 수 있는 건 아니다.**
여러 서비스가 제각각 작은 INSERT를 던지거나, 배칭 로직을 넣기 어려운 환경이 있다.
그때 두 번째 길이 필요하다.

## 길 2 — async insert (배칭 책임을 서버로 넘긴다)

`async_insert`를 켜면, **배칭을 클라이언트 대신 서버가 한다.**
들어오는 작은 INSERT들을 ClickHouse가 **서버 측 인메모리 버퍼**에 모았다가, 한꺼번에 flush해서 통합 part로 만든다.
클라이언트는 평소처럼 작은 INSERT를 던지기만 하면 되고, part 폭발은 서버가 막아준다.

```sql
-- 서버측 배칭 + 디스크 flush 후 ack (내구성 보장)
SET async_insert = 1;
SET wait_for_async_insert = 1;

INSERT INTO events (ts, user_id, payload) VALUES (...);
-- 여러 클라이언트의 작은 INSERT가 서버 버퍼에 모여
-- 아래 세 조건 중 먼저 도달하는 것에서 하나의 part로 flush됨
```

### 버퍼는 언제 비워지나

async insert 버퍼는 **세 조건 중 먼저 도달하는 것**에서 flush된다(공식 문서 기준).

| 조건 | 설정 | 의미 |
|---|---|---|
| 데이터 크기 | `async_insert_max_data_size` | 버퍼가 일정 용량 차면 flush |
| 시간 경과 | `async_insert_busy_timeout_ms` | 일정 시간 지나면 flush (Cloud 기본 1000ms) |
| 쿼리 수 | `async_insert_max_query_number` (기본 450) | 누적 INSERT 수가 차면 flush |

세 가지를 함께 두는 이유는 단순하다.
유입이 빠르면 크기/쿼리 수 조건이 먼저 차서 자주 flush되고, 유입이 드물면 시간 조건이 먼저 차서 너무 오래 안 묶이게 막는다.
어느 쪽이든 part는 통합돼서 떨어진다.

(`async_insert_max_data_size`의 정확한 기본값은 공개 출처 간 차이가 있어 — 100 MiB로 적힌 곳과 10 MiB로 적힌 곳이 갈린다 — 여기선 단정하지 않는다.
실제 운영 시엔 본인 버전의 설정값을 직접 확인하는 게 안전하다.)

### 24.2부터는 타임아웃이 적응형

고정 간격으로 flush하면, 유입이 폭주할 땐 너무 자주 끊기고 한산할 땐 너무 오래 기다린다.
그래서 24.2부터 `async_insert_use_adaptive_busy_timeout`가 **기본 활성**이다(공식 문서 기준).
유입 속도에 따라 flush 간격을 동적으로 조정한다.

```sql
-- 유입 속도에 따라 50ms~1000ms 사이로 동적 조정
SET async_insert_use_adaptive_busy_timeout = 1;     -- 24.2+ 기본 활성
SET async_insert_busy_timeout_min_ms = 50;          -- 하한
SET async_insert_busy_timeout_max_ms = 1000;        -- 상한(Cloud 기본)
SET async_insert_max_query_number = 450;            -- 누적 쿼리 수 임계
-- 고빈도 유입 시 지연↓, 드문 유입 시 더 큰 배치 누적
```

범위는 하한 `..._min_ms` 50ms, 상한 `..._max_ms` 200ms(Cloud는 1000ms)다.
빠르게 들어오면 지연을 줄여 part를 자주 떨구고, 드물게 들어오면 더 모아서 큰 part로 만든다.

### 🔴 wait_for_async_insert 는 켜둬라

async insert에서 가장 중요한 설정이 `wait_for_async_insert`다.
이건 **언제 클라이언트에게 "성공했다"고 답하느냐**를 정한다(공식 문서 기준).

| 값 | 동작 | 위험 |
|---|---|---|
| **1 (기본)** | 데이터가 **디스크에 flush된 뒤**에만 ack | 안전. 에러를 즉시 돌려받음 |
| 0 | 버퍼에 담는 즉시 ack (fire-and-forget) | flush 중 에러를 클라이언트가 모름 |

`wait_for_async_insert=0`은 버퍼링되자마자 "성공"이라고 답한다.
빠르지만, 이후 flush 단계에서 실패해도 클라이언트는 알 수가 없다.
게다가 서버가 쓰기를 늦춰야 하는 상황인데도 클라이언트가 계속 빠르게 밀어넣으면 과부하가 난다.
공식 문서가 이걸 **"매우 위험(very risky)"**하다고 명시한다 — 데이터 유실을 감수할 수 있을 때만 쓰라는 뜻이다.

그래서 **프로덕션 권장 조합은 명확하다**(공식 문서 기준).

> `async_insert = 1` + `wait_for_async_insert = 1`

서버가 배칭해주는 편의는 얻으면서, 디스크 내구성과 즉각적인 에러 반환은 그대로 지킨다.

(참고로 네트워크 재시도 등으로 같은 비동기 삽입이 중복으로 들어오는 걸 막는 `async_insert_deduplicate` 같은 기능도 있지만, 기본값·정확한 동작은 공개 출처에서 확실히 확인하지 못해 여기선 "그런 옵션이 있다" 정도로만 둔다.)

## 숫자로 보는 차이 — 동기 vs async

ClickHouse가 직접 돌린 벤치마크가 둘의 차이를 보여준다(공식 블로그 기준).

| 방식 | part 거동 | 결과 |
|---|---|---|
| **동기 삽입** (10초당 200건, 각 1 part) | 총 part가 ~30,000까지 폭증 | **5분 내 임계 초과로 실패** |
| **async insert** (1초 flush 간격) | active part 8 미만, 총 part 1,300 미만 | 200~500 클라이언트에서도 **안정 유지**, CPU도 훨씬 낮음 |

추세가 분명하다.
동기로 작게 자주 넣으면 part가 터지고, async로 서버에 배칭을 맡기면 part 수가 통제된다.
(이 벤치마크 글이 임계로 '300'을 인용하는데, 이는 작성 시점·설명용 기준이고 현행 소스 기본값은 3000이다.
그래도 "동기=폭발, async=안정"이라는 결론은 유효하다.)

## 길 3 — Buffer 테이블 (차선책, 제약 많음)

클라이언트 배칭도 안 되고 async insert도 여의치 않을 때를 위한 **차선책**이 Buffer 테이블 엔진이다.
여러 서버에서 작은 INSERT가 쏟아지고 사전 배칭이 불가능한 상황에 한정해 쓴다.

Buffer 테이블은 데이터를 **RAM에 임시로 모았다가** 주기적으로 실제 대상 테이블로 flush한다.
조회는 버퍼와 대상 테이블을 합쳐서 읽는다.

```sql
-- 대상: events (MergeTree)
CREATE TABLE events_buffer AS events
ENGINE = Buffer(
  currentDatabase(), events,
  16,            -- num_layers (병렬 버퍼)
  10, 60,        -- min_time / max_time (초)
  10000, 100000, -- min_rows / max_rows
  10000000, 100000000  -- min_bytes / max_bytes
);
-- 삽입은 events_buffer로, 조회는 둘 다 합쳐서 읽힘
```

flush 조건은 "모든 min 조건을 충족" 하거나 "max 조건 중 하나라도 충족"하면 비운다.

그런데 Buffer는 제약이 많다(공식 문서 기준).
함부로 쓰면 안 된다.

- **🔴 서버가 비정상 종료되면 RAM 버퍼 데이터가 유실된다.** 내구성이 중요한 데이터엔 권장되지 않는다.
- 버퍼엔 **인덱스가 없다** — 버퍼 구간은 풀스캔된다.
- 대상 테이블로 들어가는 **삽입 순서가 뒤바뀔 수 있다** — 순서에 의존하는 `CollapsingMergeTree` 등엔 부적합.
- 복제 테이블에선 part 순서·크기가 변동돼 **중복 제거(dedup)가 동작하지 않는다.**
- `FINAL`, `SAMPLE`을 지원하지 않는다.

그래서 우선순위는 분명하다.
**클라이언트 배칭 → async insert(+wait)** 가 먼저고, Buffer는 그 둘이 다 막혔을 때의 마지막 카드다.

## 한 장으로 정리

| 상황 | 권장 패턴 |
|---|---|
| 클라이언트가 배칭 가능 | **큰 배치 동기 삽입** — 1~2초당 1회, 1만~50만 행 (최선) |
| 클라이언트 배칭 어려움 | **async insert = 1 + wait_for_async_insert = 1** — 서버측 배칭 |
| 다중 서버·배칭 불가 | **Buffer 테이블** (차선책, RAM 유실·인덱스 없음 등 제약) |
| 절대 하지 말 것 | 행 단위/초당 수백 번 작은 INSERT → Too Many Parts |

| 핵심 임계 | 값 |
|---|---|
| `parts_to_delay_insert` | 1000 (삽입 지연 시작) |
| `parts_to_throw_insert` | 3000 (삽입 거부) |
| 권장 최소 배치 | 1,000행 |
| 권장 최적 배치 | 1만~10만 행 |

여기까지가 "어떻게 안 터지게 넣느냐"다.
큰 배치로 넣든 async로 넣든, 데이터는 이제 part로 잘 쌓였다.
그런데 막상 **읽을 때** — 수십억 행에서 조건에 맞는 것만 빠르게 골라내려면 또 다른 기술이 필요하다.

다음 11장에서는 **쿼리 최적화**, 특히 `PREWHERE`로 "읽기 전에 거르는" ClickHouse 특유의 기법을 본다.
삽입을 잘했으니, 이제 잘 읽을 차례다.

---

## 참고

- ClickHouse 공식 문서 — Too Many Parts 예외(권장 배치·해결 우선순위): https://clickhouse.com/docs/knowledgebase/exception-too-many-parts
- ClickHouse 공식 문서 — Asynchronous Inserts(async_insert·wait·적응형 타임아웃): https://clickhouse.com/docs/optimize/asynchronous-inserts
- ClickHouse 공식 블로그 — Asynchronous data inserts in ClickHouse(벤치마크·작은 삽입 비용): https://clickhouse.com/blog/asynchronous-data-inserts-in-clickhouse
- ClickHouse 공식 문서 — Buffer Table Engine(문법·제약·유실 경고): https://clickhouse.com/docs/engines/table-engines/special/buffer
- ClickHouse 공식 문서 — Common getting started issues(최소/최적 배치 크기): https://clickhouse.com/blog/common-getting-started-issues-with-clickhouse
- ClickHouse/ClickHouse 소스 (MergeTreeSettings.cpp·delayInsertOrThrowIfNeeded, 기본값 1000/3000) — DeepWiki: https://deepwiki.com/ClickHouse/ClickHouse
