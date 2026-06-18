---
title: "ClickHouse 완전 정복 12장 — TTL과 저장 관리: 데이터를 나이 들게 하는 법"
date: 2026-06-13 23:47:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "운영"]
---
분석용 데이터베이스는 한 가지 숙명을 안고 있다.
데이터가 끝없이 쌓인다는 것.
로그, 이벤트, 메트릭은 매일 들어오는데 지우는 사람은 없다.
1년 전 이벤트와 어제 이벤트가 같은 빠른 디스크 위에서 같은 비용으로 자리를 차지한다.
이건 낭비다.

그래서 ClickHouse는 데이터에 "나이"를 부여하는 장치를 갖고 있다.
바로 **TTL**(Time To Live)이다.
TTL은 데이터가 일정 시간을 넘기면 자동으로 지우거나, 더 싼 디스크로 옮기거나, 요약본으로 압축하거나, 더 강하게 재압축하게 한다.
이번 장은 이 TTL과 그 아래에서 데이터를 받쳐 주는 **저장 계층(storage policy)**, 그리고 **압축 코덱**을 다룬다.

3장에서 본 것처럼 MergeTree의 데이터는 **part**(파트)라는 불변 덩어리로 저장된다.
part는 한 번 만들어지면 내용이 바뀌지 않고, 백그라운드 머지로 새 part가 만들어지고 옛 part가 버려질 뿐이다.
이 머지 메커니즘이 TTL의 동력원이다.
이 점을 기억해 두면 이번 장이 훨씬 쉽게 읽힌다.

## TTL을 한마디로 — "유통기한 스티커"

비유를 하나 던지자.
냉장고에 들어가는 모든 식재료에 유통기한 스티커를 붙인다고 생각해 보자.
어떤 건 기한이 지나면 버린다.
어떤 건 신선실(빠른 칸)에서 일반 칸으로, 다시 냉동실로 옮긴다.
어떤 건 며칠 지나면 통째로 두지 않고 "이번 주 합계"로 묶어 메모만 남긴다.

이건 어디까지나 이해를 돕는 비유다.
ClickHouse가 실제로 식재료를 옮기는 건 아니고, part를 머지하면서 만료 규칙을 적용한다.
중요한 건 TTL이 단순히 "삭제"만 하는 도구가 아니라는 점이다.
공식 가이드는 TTL의 용도를 세 갈래로 정리한다 (공식 문서 기준).
오래된 데이터 제거, 디스크 간 데이터 이동, 데이터 롤업(요약).

TTL 표현식은 시간 타입(`Date` / `DateTime` 계열)으로 평가돼야 하고, `INTERVAL` 연산자를 쓴다.
예를 들어 `event_time + INTERVAL 1 MONTH`은 "이벤트 시각으로부터 한 달 뒤"라는 만료 시점을 뜻한다.

## TTL이 할 수 있는 일들 — 한눈에

테이블 레벨 TTL의 전체 문법은 한 절 안에 여러 규칙을 콤마로 나열할 수 있게 돼 있다.
기본 동작은 `DELETE`이며, 그 외에 이동·롤업·재압축을 붙일 수 있다.

| 동작 | 문법 | 하는 일 |
|---|---|---|
| 삭제 | `TTL expr DELETE` | 만료된 행 제거 (기본 동작) |
| 조건부 삭제 | `TTL expr DELETE WHERE cond` | 조건 만족하는 만료 행만 삭제 |
| 디스크 이동 | `TTL expr TO DISK 'x'` | 만료 part를 특정 디스크로 이동 |
| 볼륨 이동 | `TTL expr TO VOLUME 'x'` | 만료 part를 특정 볼륨(계층)으로 이동 |
| 롤업 | `TTL expr GROUP BY k SET ...` | 삭제 전 집계해 요약본으로 압축 |
| 재압축 | `TTL expr RECOMPRESS CODEC(...)` | 만료 시점에 다른 코덱으로 재압축 |
| 컬럼 만료 | 컬럼에 `col T TTL expr` | 특정 컬럼만 만료 → 기본값으로 |

하나씩 보자.

### 삭제와 조건부 삭제

가장 단순한 형태다.
`TTL d + INTERVAL 1 YEAR DELETE`는 1년 지난 행을 지운다.
재미있는 건 조건을 붙일 수 있다는 점이다.
이벤트 종류별로 보존 기간을 다르게 주고 싶을 때 쓴다.

```sql
TTL time + INTERVAL 1 MONTH DELETE WHERE event != 'error',
    time + INTERVAL 6 MONTH DELETE WHERE event = 'error'
```

일반 이벤트는 한 달만, 에러 이벤트는 여섯 달 보관.
규제나 디버깅 때문에 "에러만 오래 남기고 싶다"는 흔한 요구를 SQL 한 줄로 푼다.

### 컬럼 단위 만료

행이 아니라 특정 컬럼만 만료시킬 수도 있다.
컬럼 TTL이 발동하면 해당 값은 그 타입의 기본값으로 바뀐다.
그리고 한 part 안에서 그 컬럼 값이 전부 만료되면, 그 컬럼 파일은 디스크에서 통째로 사라진다 (공식 문서 기준).

```sql
ALTER TABLE tab MODIFY COLUMN c String TTL d + INTERVAL 1 DAY
```

쓸 만한 경우는 이렇다.
원본 페이로드 같은 무거운 컬럼은 하루만 두고 비우되, 집계에 필요한 가벼운 컬럼은 오래 남긴다.
단, 정렬키/기본키에 들어간 컬럼에는 컬럼 TTL을 걸 수 없다.
그 컬럼이 사라지면 4장에서 본 sparse primary index 자체가 무너지기 때문이라고 이해하면 자연스럽다.

### 롤업(GROUP BY) — 버리기 전에 요약하기

이게 TTL에서 가장 강력한 기능이다.
원본 행을 삭제하기 직전에 집계해서, 상세 데이터는 버리되 요약본은 남긴다.

```sql
CREATE TABLE hits_rollup
(
    timestamp DateTime,
    id String,
    hits Int32,
    max_hits Int32 DEFAULT hits,
    sum_hits Int64 DEFAULT hits
)
ENGINE = MergeTree
PRIMARY KEY (id, toStartOfDay(timestamp), timestamp)
TTL timestamp + INTERVAL 1 DAY
    GROUP BY id, toStartOfDay(timestamp)
    SET max_hits = max(max_hits),
        sum_hits = sum(sum_hits);
```

하루가 지나면 같은 `id`의 같은 날 데이터가 한 행으로 압축되고, 그 안에 최대값과 합계가 남는다.
여기서 꼭 지켜야 할 규칙이 있다.
**`GROUP BY` 컬럼은 반드시 테이블 `PRIMARY KEY`의 접두(prefix)여야 한다** (공식 문서 기준).
위 예에서 키가 `(id, toStartOfDay(timestamp), timestamp)`이고 `GROUP BY id, toStartOfDay(timestamp)`인 것이 그 이유다.
그리고 집계 대상 컬럼에는 `DEFAULT` 값이 필요하다.

### 재압축(RECOMPRESS)

데이터가 식으면 더 강하게 눌러 담는다.
최근 데이터는 빠른 LZ4로, 오래된 콜드 데이터는 시간이 좀 걸려도 ZSTD 고압축으로 바꿔 디스크를 아낀다.

```sql
TTL d + INTERVAL 1 MONTH RECOMPRESS CODEC(ZSTD(17)),
    d + INTERVAL 1 YEAR  RECOMPRESS CODEC(LZ4HC(10))
```

압축률과 CPU 비용은 맞바꿈 관계다.
자주 안 읽는 오래된 데이터일수록 압축에 CPU를 더 써도 손해가 적으니, 나이에 따라 코덱을 바꾸는 건 합리적인 전략이다.

## 계층형 저장 — hot/warm/cold

TTL의 `TO VOLUME` / `TO DISK`는 part를 "다른 저장 계층"으로 옮긴다.
빠르지만 비싼 SSD(hot)에서, 느리지만 싼 HDD나 S3(cold)로.
데이터가 나이 들수록 더 싼 곳으로 흘려보내는 것이다.

이게 동작하려면 먼저 ClickHouse에 "어떤 저장소가 있는지"를 알려 줘야 한다.

![](/assets/img/posts/clickhouse-12-ttl-저장관리/clickhouse-tiered-ttl.svg)

이 구조는 세 층으로 돼 있다 (공식 문서 기준).

| 개념 | 의미 |
|---|---|
| disk | 파일시스템에 마운트된 실제 블록 디바이스 |
| volume | 같은 성격 디스크들의 정렬된 묶음 (JBOD와 유사) |
| policy | 볼륨들의 집합 + 그 사이 이동 규칙 |

`system.disks`, `system.storage_policies` 테이블로 현재 설정을 조회할 수 있다.
테이블에는 `SETTINGS storage_policy = 'tiered'`처럼 정책을 지정해 적용한다.

### 자동 이동: move_factor

흥미로운 건 TTL 시간 규칙 없이도 데이터가 자동으로 옮겨진다는 점이다.
열쇠는 **`move_factor`**다.
한 볼륨의 가용 공간 비율이 이 값 아래로 떨어지면, ClickHouse가 part를 다음 볼륨으로 자동으로 흘려보낸다.
기본값은 0.1이다 (공식 문서 기준).

| 파라미터 | 역할 |
|---|---|
| `move_factor` | 가용 공간이 이 비율 미만이면 다음 볼륨으로 이동 (기본 0.1) |
| `max_data_part_size_bytes` | 볼륨이 받을 최대 part 크기 (초과분은 다음 볼륨으로) |
| `volume_priority` | 채우는 우선순위 (낮을수록 먼저) |
| `load_balancing` | `round_robin` 또는 `least_used` |
| `keep_free_space_bytes` | 예약해 둘 빈 공간 |

비유하자면 `move_factor`는 신선실이 80% 차면 자동으로 일반 칸으로 밀어내는 센서다.
`max_data_part_size_bytes`는 더 직접적이다.
큰 part는 자동으로 더 큰 다음 볼륨으로 라우팅된다 — 작은 최신 데이터는 빠른 디스크, 큰 머지 결과물은 큰 디스크로 가는 계층 분리의 핵심이다.

이동에 관해 알아 둘 함정이 둘 있다.
mutation과 partition freeze는 hard link를 쓰기 때문에 디스크 간 이동을 하지 못한다 — part가 원래 디스크에 남는다.
그리고 이동 작업의 스레드 풀 크기는 `background_move_pool_size`로 제어한다.

## 언제 TTL이 실제로 실행되나

여기가 운영에서 가장 헷갈리는 지점이다.
"`TTL ... 1 DAY DELETE`를 걸었는데 왜 어제 데이터가 아직 있지?"

답은 단순하다.
**TTL은 주로 백그라운드 머지 중에 평가·적용된다** (공식 문서 기준).
insert 시점에는 적용되지 않는다.
즉 만료 시각이 지났어도, 그 part가 다음에 머지될 때까지는 데이터가 물리적으로 남아 있을 수 있다.

여기엔 예외가 하나 있다.
**move TTL**은 insert 시점에 즉시 적용될 수 있다.
들어오는 part가 이미 이동 규칙을 만족하면, 곧장 대상 볼륨에 쓰는 것이다.
이 동작을 켜고 끄는 게 `perform_ttl_move_on_insert`이고, 기본값은 1(활성)이다 (공식 소스 확인 기준).
끄면(0) part를 일단 기본 볼륨에 쓰고 백그라운드 태스크가 나중에 옮긴다.

삭제가 너무 늦게 일어나는 게 문제라면, ClickHouse는 다음 정기 머지를 무작정 기다리지 않고 만료 데이터를 감지하면 별도 머지를 트리거할 수도 있다.
그래도 즉시 정리를 강제하고 싶으면 이 한 줄이 확실하다.

```sql
OPTIMIZE TABLE events_tiered FINAL;
```

이건 강제 머지라 비용이 크니 운영에선 남발하지 말 것.

머지 주기를 조절하는 설정도 있다.
삭제 머지는 `merge_with_ttl_timeout`, 재압축·롤업 머지는 `merge_with_recompression_ttl_timeout`으로 제어한다.
공개적으로 확인되는 범위에선 TTL 가이드 문서가 두 값의 기본을 각각 14400초(4시간)로 적고 있다.
다만 설정 페이지에는 기본값이 명시돼 있지 않아, 정확한 수치는 쓰는 버전에서 한 번 확인하길 권한다.

⚠️ 한 가지 주의.
TTL 표현식에 `now()`, `rand()` 같은 비결정 함수를 쓰면 안 된다.
머지가 일어날 때마다 값이 다시 계산돼서, 어떤 행이 언제 삭제될지 예측할 수 없게 된다.

## 효율의 핵심: ttl_only_drop_parts와 파티션 정렬

TTL 삭제가 비효율적일 때가 있다.
한 part 안에 만료된 행과 안 만료된 행이 섞여 있으면, ClickHouse는 살아남을 행만 골라 part를 통째로 다시 써야 한다.
대용량에서 이 재작성 I/O는 만만치 않다.

그래서 공식이 권장하는 패턴이 있다 (공식 문서 기준).
**TTL에 쓰는 시간 필드의 날짜/월로 `PARTITION BY`를 맞춘다.**
그러면 한 파티션의 모든 데이터가 같은 만료 시점을 갖게 돼서, 만료된 part는 재작성 없이 통째로 버릴 수 있다.

이걸 거드는 설정이 **`ttl_only_drop_parts`**다.
part의 모든 행이 만료됐을 때 part를 행 단위로 재작성하지 않고 통째로 드롭하게 한다.
공개적으로 확인되는 범위에선 기본값은 0(비활성)이며, 1로 켜는 것이 위 파티션 정렬과 짝을 이루는 운영 베스트 프랙티스다.

아래는 이 모든 걸 합친 예다.
계층 이동 + 삭제 + 파티션 정렬 + 통째 드롭.

```sql
-- storage policy 'tiered' 가 disks(fast_ssd, hdd)와
-- volumes(hot, cold)로 정의돼 있다고 가정
CREATE TABLE events_tiered
(
    d DateTime,
    event String,
    value UInt64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(d)
ORDER BY (event, d)
TTL d + INTERVAL 1 WEEK  TO VOLUME 'hot',   -- 최근 1주: 빠른 SSD
    d + INTERVAL 1 MONTH TO VOLUME 'cold',  -- 1개월~: 느린 HDD/S3
    d + INTERVAL 1 YEAR  DELETE             -- 1년 후 삭제
SETTINGS storage_policy = 'tiered',
         ttl_only_drop_parts = 1;          -- 파티션 정렬 시 통째 드롭
```

기존 테이블에 TTL을 바꿨다면, 과거 데이터엔 자동 소급되지 않는다.
`ALTER TABLE t MATERIALIZE TTL`로 기존 데이터에 규칙을 적용해 줘야 한다.

## 스토리지/컴퓨트 분리 — S3에 데이터를 둔다

계층의 끝에는 보통 객체 스토리지가 온다.
ClickHouse는 `type=s3` 디스크로 데이터를 S3에 두고 컴퓨트와 분리할 수 있다 (공식 문서 기준).
설정 파일에 s3 디스크(와 선택적으로 캐시 디스크)를 정의하고 `storage_policy`로 테이블에 붙이면 된다.

```sql
<clickhouse>
  <storage_configuration>
    <disks>
      <s3_disk>
        <type>s3</type>
        <endpoint>$BUCKET</endpoint>
        <access_key_id>$ACCESS_KEY_ID</access_key_id>
        <secret_access_key>$SECRET_ACCESS_KEY</secret_access_key>
        <metadata_path>/var/lib/clickhouse/disks/s3_disk/</metadata_path>
      </s3_disk>
      <s3_cache>
        <type>cache</type>
        <disk>s3_disk</disk>
        <path>/var/lib/clickhouse/disks/s3_cache/</path>
        <max_size>10Gi</max_size>
      </s3_cache>
    </disks>
    <policies>
      <s3_main>
        <volumes><main><disk>s3_disk</disk></main></volumes>
      </s3_main>
    </policies>
  </storage_configuration>
</clickhouse>
-- 사용: CREATE TABLE t (...) ENGINE=MergeTree ORDER BY id
--       SETTINGS storage_policy='s3_main';
```

여기서 `s3_cache`는 S3 위에 얹는 로컬 읽기 캐시다.
객체 스토리지의 읽기 지연을 로컬 캐시로 메워 준다.
로컬 SSD를 hot, S3를 cold로 한 정책에 묶으면 자연스러운 hot→cold 계층이 완성된다.

🔴 꼭 기억할 주의 하나.
S3 디스크를 쓸 때 **AWS/GCS 라이프사이클 정책을 설정하면 안 된다** (공식 문서 기준).
ClickHouse가 지원하지 않으며 테이블이 손상될 수 있다.
데이터의 수명 관리는 라이프사이클이 아니라 ClickHouse의 TTL에 맡겨야 한다.
그리고 S3 디스크의 내결함성은 8장에서 본 `ReplicatedMergeTree`로 확보한다.

## 압축 코덱 — 디스크를 줄이는 또 한 축

저장 관리의 마지막 축은 압축이다.
2장에서 컬럼 지향이 왜 압축에 유리한지 봤다 — 같은 컬럼 값들이 줄지어 있으니 패턴이 잘 잡힌다.
ClickHouse는 컬럼마다 `CODEC()`으로 압축 방식을 지정할 수 있다 (공식 문서 기준).

| 종류 | 코덱 | 쓰임 |
|---|---|---|
| 범용 | LZ4 | 셀프매니지드 기본, 빠름 |
| 범용 | LZ4HC | LZ4 고압축 (레벨 1~12, 기본 9) |
| 범용 | ZSTD | 강한 압축 (레벨 1~22, 기본 1), Cloud 기본 |
| 특화 | Delta / DoubleDelta | 천천히 증가하는 정수 (타임스탬프 등) |
| 특화 | Gorilla / FPC | 느리게 변하는 float |
| 특화 | T64 / GCD | 정수 비트 패킹 / 최대공약수 |

특화 코덱은 범용 코덱과 조합해서 쓸 수 있다.
특화 코덱이 데이터를 전처리한 뒤 범용 코덱이 최종 압축하는 식이다.

```sql
CREATE TABLE metrics
(
    d DateTime CODEC(DoubleDelta),       -- 시계열 타임스탬프
    sensor_value Float32 CODEC(Gorilla), -- 느리게 변하는 float
    payload String CODEC(LZ4)            -- 기본 빠른 압축
)
ENGINE = MergeTree
ORDER BY d
TTL d + INTERVAL 1 MONTH RECOMPRESS CODEC(ZSTD(17)),
    d + INTERVAL 1 YEAR  RECOMPRESS CODEC(LZ4HC(10));
```

타임스탬프처럼 일정 간격으로 늘어나는 값은 `DoubleDelta`로 차이의 차이만 저장하면 거의 0에 수렴한다.
센서값처럼 비슷한 값이 이어지는 float은 `Gorilla`가 잘 맞는다.
여기에 앞서 본 `RECOMPRESS` TTL을 붙이면, 시간이 흐를수록 콜드 데이터가 더 강한 코덱으로 다시 눌린다.
**코덱(공간 효율)과 TTL(시간 관리)이 만나는 지점**이 바로 이 패턴이다.

## 왜 이렇게 설계했을까

마지막으로 한 번 묻자.
ClickHouse는 왜 삭제를 즉시 하지 않고 머지에 얹었을까.

답은 3장의 철학과 같다.
part는 불변이고, 모든 변화는 백그라운드 머지를 통한다.
삭제·이동·롤업·재압축을 머지에 얹으면, 어차피 일어날 머지에 일을 하나 더 붙이는 셈이라 추가 I/O를 최소화할 수 있다.
대신 "즉시"는 포기한다 — 만료가 곧바로 반영되지 않는 트레이드오프를 받아들인 것이다.

그래서 운영의 정석은 이렇게 정리된다.
삭제 목적이면 시간 필드로 파티셔닝하고 `ttl_only_drop_parts=1`로 part를 통째 드롭한다.
보존이 길면 계층형 저장으로 오래된 데이터를 싼 디스크/S3로 흘려보낸다.
공간이 더 필요하면 콜드 데이터를 `RECOMPRESS`로 강하게 다시 압축한다.
즉시 정리가 필요한 예외 상황에서만 `OPTIMIZE ... FINAL`을 쓴다.

다음 13장에서는 이렇게 굴러가는 머지·이동·TTL이 실제로 어떻게 돌고 있는지 들여다보는 법 — `system` 테이블 기반의 운영·모니터링을 다룬다.
`system.parts`, `system.merges` 같은 창으로 클러스터 속을 직접 보는 장이다.

## 참고

- TTL for Managing Data — https://clickhouse.com/docs/guides/developer/ttl
- MergeTree Table Engine (TTL · storage policy · disks/volumes) — https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree
- Separation of Storage and Compute (S3 disk) — https://clickhouse.com/docs/guides/separation-storage-compute
- CREATE TABLE — Column Compression Codecs — https://clickhouse.com/docs/sql-reference/statements/create/table
- MergeTree Settings — https://clickhouse.com/docs/operations/settings/merge-tree-settings
