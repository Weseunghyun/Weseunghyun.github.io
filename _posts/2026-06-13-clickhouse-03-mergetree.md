---
title: "ClickHouse 완전 정복 3장 — MergeTree: parts·ORDER BY·백그라운드 머지"
date: 2026-06-13 23:56:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "핵심"]
---
2장에서 ClickHouse가 빠른 네 가지 이유를 봤다.
그런데 그 모든 게 실제로 **어디에, 어떻게** 저장되느냐는 아직 안 봤다.
그 답이 **MergeTree**다 — ClickHouse의 **기본이자 심장인 저장 엔진**.
ClickHouse에서 "테이블을 만든다"는 건 거의 항상 "MergeTree 계열 엔진으로 테이블을 만든다"는 뜻이다.

이 장에서 세 가지를 잡는다.

1. **part** — 데이터가 저장되는 덩어리.
   그리고 이게 **불변(immutable)**이라는 점.
2. **ORDER BY** — MergeTree의 모든 것을 좌우하는 정렬키.
3. **백그라운드 머지** — "Merge"Tree라는 이름의 정체.

## 먼저, 테이블 하나를 만들어 보자

```sql
CREATE TABLE events
(
    event_time  DateTime,
    user_id     UInt64,
    event_type  String,
    url         String
)
ENGINE = MergeTree
ORDER BY (event_time, user_id);
```

`ENGINE = MergeTree`가 "이 테이블은 MergeTree 방식으로 저장한다"는 선언이고,
`ORDER BY (event_time, user_id)`가 **데이터를 무엇으로 정렬해 둘지**를 정한다.
이 `ORDER BY`가 이 장에서 가장 중요한 한 줄이다 — 왜 그런지는 곧 본다.

## 1. part — 삽입은 "새 덩어리"를 만든다

ClickHouse에 데이터를 `INSERT`하면 무슨 일이 일어날까?
일반 DB처럼 "테이블에 행을 끼워 넣는" 게 아니다.
ClickHouse는 들어온 데이터를 **`ORDER BY` 키로 정렬한 뒤, 새로운 part(파트)를 통째로 디스크에 쓴다**(소스 기준).

> **part = MergeTree에서 데이터가 저장되는 기본 단위.** 한 번의 삽입(또는 머지)이 하나의 part를 만든다.

part 하나는 디렉터리 하나라고 생각하면 된다.
그 안에 컬럼 데이터, 인덱스(4장), 마크 파일 등이 들어 있다.
part의 내부 포맷은 둘 중 하나다(소스 기준).

| 포맷 | 저장 방식 | 언제 |
|---|---|---|
| **Wide** | 컬럼마다 **별도 파일** | part가 클 때 (기본 다수) |
| **Compact** | 모든 컬럼을 **한 파일**에 | part가 작을 때 |

어느 쪽이 될지는 `min_bytes_for_wide_part`·`min_rows_for_wide_part` 설정으로 갈린다.
작은 part를 Compact로 두는 건, 파일이 너무 잘게 쪼개지는 걸 막기 위함이다.

### 🔴 part는 불변(immutable)이다 — 이게 모든 것을 설명한다

MergeTree에서 가장 중요한 성질 하나를 꼽으라면 이것이다.

> **part는 불변이다. 한 번 쓰이면 절대 수정되지 않는다. 오직 생성되거나 삭제될 뿐이다.** (소스 기준)

이 한 문장이 ClickHouse의 동작 대부분을 설명한다.

- **삽입이 빠른 이유**: 기존 데이터를 건드리지 않고 새 part만 쓰면 되니, 기존 구조와 동기화할 필요가 없다.
  공식 문서 표현으로 삽입은 **"거의 디스크 I/O 속도로"** 일어나고, 진행 중인 `SELECT`와 완전히 격리된다.
- **`UPDATE`/`DELETE`가 무거운 이유**(1장): part를 고칠 수 없으니, 수정은 **관련 part를 통째로 다시 쓰는** mutation으로 처리된다.
- **머지가 필요한 이유**: 삽입마다 part가 생기면 part가 끝없이 늘어난다.
  그래서 합쳐야 한다(아래).

> 비유: part는 **잉크로 적어 코팅해버린 기록판** 같다. 한 번 굳으면 고쳐 쓸 수 없다. 내용을 바꾸려면 새 판을 만들어 갈아끼우는 수밖에 없다. 대신 굳히는(=쓰는) 건 아주 빠르고, 누가 그 판을 읽고 있어도 방해하지 않는다.

## 2. ORDER BY — MergeTree의 모든 것을 좌우하는 한 줄

`ORDER BY`는 단순히 "정렬해서 보여줘"가 아니다.
MergeTree에서 `ORDER BY`는 **데이터가 디스크에 물리적으로 어떤 순서로 놓이는지**를 정한다.
그리고 그 물리적 순서가 세 가지를 동시에 결정한다.

1. **정렬키이자 기본키(primary key)** — `PRIMARY KEY`를 따로 안 쓰면, `ORDER BY`가 곧 기본키가 된다(소스 기준).
   이 기본키로 sparse 인덱스가 만들어진다(4장).
2. **압축률** — 2장에서 봤듯, 비슷한 값이 인접하면 압축이 잘 된다.
   `ORDER BY`가 그 "인접"을 만든다.
   공식 문서 실측으로 정렬 순서만 바꿔 압축비가 **3:1 → 39:1**로 좋아진 예가 있다.
3. **쿼리 스킵 효율** — `ORDER BY`로 정렬돼 있어야, "지난 1시간"을 물을 때 그 범위가 든 부분만 골라 읽고 나머지를 건너뛸 수 있다(4장 sparse 인덱스).

그래서 ClickHouse 테이블 설계의 **8할이 `ORDER BY`를 잘 잡는 일**이다.
대원칙 하나만 미리 말해두면(자세한 건 4·5장):

> **자주 필터·정렬하는 컬럼을, 카디널리티(값 종류 수)가 낮은 것부터** `ORDER BY`에 둔다.

이걸 어기면 압축도 나빠지고 쿼리도 느려진다.
4장에서 이 원칙이 왜 그런지를 sparse 인덱스로 증명한다.

## 3. 백그라운드 머지 — "Merge"Tree라는 이름의 정체

이제 마지막 조각이자 이름의 유래다.
삽입마다 새 part가 생긴다고 했다.
초당 여러 번 삽입하면 part가 순식간에 수백·수천 개가 된다.
part가 많아지면 쿼리할 때 그 많은 part를 일일이 열어야 하니 **느려진다.**

그래서 ClickHouse는 백그라운드에서 **작은 part들을 모아 큰 part 하나로 합친다.**
이게 **머지(merge)**이고, 엔진 이름 **Merge**Tree가 여기서 왔다.

![](/assets/img/posts/clickhouse-03-mergetree/clickhouse-mergetree-parts.svg)

머지의 성질(소스 기준):

- **백그라운드에서** 자동으로 일어난다.
  진행 중인 `SELECT`를 막지 않는다.
- 작은 part 여러 개 → 정렬을 유지한 **큰 part 하나**.
  (각 part가 이미 `ORDER BY`로 정렬돼 있으니, 정렬된 것들을 합치는 건 빠르다.)
- part는 불변이므로, 머지는 **새 part를 만들고 옛 part들을 나중에 삭제**하는 식이다.
- **파티션이 다르면 합치지 않는다** — 머지는 같은 파티션 안에서만 일어난다(파티션은 아래).

> 비유: 삽입이 메모지를 한 장씩 쌓는 거라면, 머지는 그 메모지 묶음을 주기적으로 **한 권의 정렬된 공책으로 다시 묶는** 일이다. 묶을수록 찾기 쉬워진다. 다만 이 묶는 일이 **공짜는 아니다** — CPU와 디스크를 쓴다. 그래서 "작은 삽입을 너무 자주 하면" 머지가 따라잡지 못해 part가 쌓이는 문제가 생긴다(10·13장에서 다룬다).

### 머지 시점에 "미리 계산"까지 한다

머지는 단순히 합치기만 하는 게 아니다.
ClickHouse는 머지하는 **그 김에** 특별한 연산을 미리 해두기도 한다(공식 문서).
이게 MergeTree의 변종들이다.

| 엔진 | 머지 때 추가로 하는 일 |
|---|---|
| `MergeTree` | 합치기만 |
| `ReplacingMergeTree` | 같은 키의 중복 행을 **하나만 남김** |
| `SummingMergeTree` | 같은 키의 숫자 컬럼을 **합산** |
| `AggregatingMergeTree` | 같은 키를 **집계 함수로 미리 집계** |

이 "머지 시점에 미리 계산해두기" 덕에, 나중 쿼리는 이미 줄어든/집계된 데이터만 읽으면 된다.
공식 문서는 이 덕분에 사용자 쿼리가 **"때로 1000배 이상"** 빨라질 수 있다고 말한다.
이 변종들과 `Materialized View`의 조합이 6장의 주제다.

## (짧게) 파티션 — 머지와 삭제의 경계선

`ORDER BY`와 자주 헷갈리는 게 **파티션(`PARTITION BY`)**이다.
파티션은 데이터를 **큰 덩어리로 나누는 경계선**이다 — 보통 월·일 단위로 잡는다.

```sql
ENGINE = MergeTree
PARTITION BY toYYYYMM(event_time)   -- 월별로 파티션
ORDER BY (event_time, user_id);
```

- 머지는 **같은 파티션 안에서만** 일어난다.
- 그래서 "2025년 1월 데이터 통째 삭제" 같은 작업이 **파티션 단위로 빠르게** 된다(그 파티션 part들만 버리면 됨).
- 단, 파티션을 너무 잘게 쪼개면(예: 분 단위) part가 과하게 많아져 오히려 느려진다 — 흔한 안티패턴이다.

`ORDER BY`가 "파티션 **안에서** 어떻게 정렬하나"라면,
`PARTITION BY`는 "데이터를 **어떤 덩어리로** 나누나"다.
역할이 다르다.

## 한 장으로 정리

| 개념 | 한 줄 |
|---|---|
| **part** | 삽입/머지가 만드는 저장 단위. **불변** — 생성·삭제만. |
| Wide / Compact | part 내부 포맷. 크면 Wide(컬럼별 파일), 작으면 Compact(한 파일). |
| **ORDER BY** | 물리 정렬 순서 = 기본키. 압축·쿼리 스킵·인덱스를 동시에 좌우. |
| **백그라운드 머지** | 작은 part들을 큰 part로 합침. SELECT 안 막음. 파티션 내에서만. |
| 변종(Replacing/Summing/Aggregating) | 머지 시점에 중복 제거·합산·집계를 미리 함. |
| PARTITION BY | 데이터를 큰 덩어리로 나누는 경계. 머지·삭제의 단위. |

여기까지가 "데이터가 어디에 어떻게 놓이나"다.
그런데 아직 풀지 않은 약속이 있다.
2장 끝에서 던진 질문 — "컬럼 안에서도 **필요한 행만** 골라 읽을 수 있나?"
그리고 이 장에서 미룬 것 — "`ORDER BY`로 정렬해두면 그걸 어떻게 이용해 건너뛰나?"

그 답이 ClickHouse의 가장 영리한 부분, **sparse primary index(성긴 기본 인덱스)**다.
8192행마다 딱 하나의 표시로 수십억 행을 건너뛰는 구조 — 4장에서 본다.

---

## 참고

- ClickHouse 공식 문서 — MergeTree Engine: https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree
- ClickHouse 공식 문서 — Why is ClickHouse so fast?
  (머지 시점 연산, 삽입 격리): https://clickhouse.com/docs/concepts/why-clickhouse-is-so-fast
- ClickHouse/ClickHouse 소스 (part 불변성·Wide/Compact·머지) — DeepWiki: https://deepwiki.com/ClickHouse/ClickHouse
