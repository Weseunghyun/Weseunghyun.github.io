---
title: "ClickHouse 완전 정복 9장 — 분산과 샤딩: 한 대로 안 될 때 데이터를 쪼갠다"
date: 2026-06-13 23:50:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "운영"]
---
지금까지 우리는 ClickHouse를 한 대의 서버 안에서 다뤘다.
컬럼 저장(2장), MergeTree의 part와 머지(3장), sparse primary index(4장)까지 — 전부 "한 노드 안에서 어떻게 빨리 읽느냐"는 이야기였다.
하지만 데이터가 한 대 서버의 디스크나 메모리로 감당이 안 될 만큼 커지면 어떻게 할까.
이번 장은 그 답이다.
ClickHouse가 데이터를 여러 서버에 나눠 담고(샤딩), 그 위에서 마치 한 테이블처럼 쿼리하는 방법.

## 먼저 원리부터: 데이터를 쪼개는 게 샤딩이다

샤딩(sharding)은 하나의 큰 테이블을 여러 서버에 조각내어 나눠 담는 것이다.
각 조각을 담는 서버를 샤드(shard)라고 부른다.
샤드가 2개면 데이터의 절반씩, 4개면 4분의 1씩 나눠 갖는다.

여기서 중요한 건 ClickHouse가 이걸 "두 층"으로 나눠 처리한다는 점이다.

- **로컬 테이블(local table)**: 각 샤드 안에 실제 데이터가 들어 있는 진짜 테이블.
  보통 MergeTree다(3장).
- **Distributed 테이블**: 자체 데이터는 한 줄도 없고, 여러 샤드의 로컬 테이블을 가리키는 "뷰"일 뿐인 테이블.

공식 문서는 Distributed 엔진을 이렇게 설명한다.
"Distributed 엔진 테이블은 자체 데이터를 저장하지 않으며, 여러 서버에 걸친 분산 쿼리 처리를 가능하게 한다.
읽기는 자동으로 병렬화된다"(공식 문서 기준).
즉 실제 데이터는 각 샤드의 로컬 테이블에 흩어져 있고, Distributed 테이블은 그 위에 얹힌 라우팅 계층이다.

![](/assets/img/posts/clickhouse-09-분산-샤딩-distributed/clickhouse-sharding-replication.svg)
(복제는 8장의 ReplicatedMergeTree·Keeper와 합쳐진 그림 — 샤딩으로 나누고, 각 샤드를 복제본으로 복사한다.)

## 비유 하나: 분산 창고와 안내 데스크

이해를 돕는 비유를 하나 던지겠다(어디까지나 비유다, 실제 구조는 이 장 뒤에서 정확히 본다).

거대한 물류 회사가 창고 한 동에 물건을 다 못 넣어서, 창고를 2동·3동으로 늘렸다고 하자.
각 창고(샤드)에는 실제 물건(로컬 테이블의 데이터)이 들어 있다.
그런데 손님이 "내 주문 전부 보여줘"라고 할 때 창고를 일일이 돌게 할 순 없다.
그래서 입구에 **안내 데스크 한 곳(Distributed 테이블)**을 둔다.
손님은 안내 데스크에만 말하면 되고, 데스크가 알아서 모든 창고에 동시에 연락해(읽기 병렬화) 결과를 모아 손님에게 준다.
물건을 새로 넣을 때도 데스크가 "이 물건은 어느 창고로"를 정해 보낸다.

이 비유에서 "어느 창고로 보낼지 정하는 규칙"이 바로 샤딩 키(sharding key)다.

## Distributed 엔진의 생김새

Distributed 테이블은 이렇게 만든다(공식 문서 기준 파라미터).

```sql
Distributed(cluster, database, table[, sharding_key[, policy_name]])
```

- `cluster`: 서버 설정에 정의된 클러스터 이름.
- `database`, `table`: 각 샤드에 있는 **원격 로컬 테이블** 이름.
- `sharding_key`(옵션): INSERT 시 어느 샤드로 보낼지 정하는 식.
- `policy_name`(옵션): 백그라운드 전송용 임시파일을 어디 둘지 정하는 스토리지 정책.

실제 2단 구조를 만들어 보면 이렇다.
먼저 각 샤드에 데이터를 담을 로컬 테이블을, 그리고 그 위에 진입점이 될 Distributed 테이블을 만든다.

```sql
-- 1) 각 샤드에 실제 데이터를 담는 로컬 테이블 생성
CREATE TABLE uk.uk_price_paid_local ON CLUSTER cluster_2S_1R
(
    price      UInt32,
    postcode1  LowCardinality(String),
    postcode2  LowCardinality(String),
    addr1      String,
    addr2      String
)
ENGINE = MergeTree
ORDER BY (postcode1, postcode2, addr1, addr2);

-- 2) 쿼리/쓰기 진입점이 되는 Distributed 테이블 (자체 데이터 없음)
CREATE TABLE uk.uk_price_paid_distributed ON CLUSTER cluster_2S_1R
ENGINE = Distributed('cluster_2S_1R', 'uk', 'uk_price_paid_local', rand());
```

이렇게 하면 `uk_price_paid_distributed`에 쿼리를 던지는 것만으로 두 샤드의 데이터를 합쳐서 조회할 수 있다(공식 문서 기준).
여기서 `ON CLUSTER cluster_2S_1R`은 잠시 뒤에 설명할 "분산 DDL"이다.

## 클러스터는 어디에 정의하나: remote_servers

위 코드의 `'cluster_2S_1R'`라는 이름은 어디서 왔을까.
서버 설정 파일의 `<remote_servers>` 블록이다.
여기에 클러스터 토폴로지(어떤 샤드가 있고, 각 샤드에 어떤 서버가 있는지)를 적는다.

```sql
<!-- config.xml / cluster.xml -->
<remote_servers>
  <cluster_2S_1R>
    <shard>
      <internal_replication>true</internal_replication>
      <replica><host>clickhouse-01</host><port>9000</port></replica>
    </shard>
    <shard>
      <internal_replication>true</internal_replication>
      <replica><host>clickhouse-02</host><port>9000</port></replica>
    </shard>
  </cluster_2S_1R>
</remote_servers>
```

`<shard>`가 두 개이므로 2 샤드, 각 샤드에 `<replica>`가 하나씩이므로 1 레플리카(복제본 없음) 구성이다.
그래서 이름이 `cluster_2S_1R`(2 Shards, 1 Replica)이다.
공식 문서는 이 설정이 "ON CLUSTER 절을 사용하는 분산 DDL 쿼리의 템플릿 역할을 한다"고 설명한다(공식 문서 기준).

## ON CLUSTER: 한 번의 DDL을 모든 노드에

샤드가 여러 대라면, 테이블 하나 만들 때마다 모든 서버에 일일이 접속해 같은 CREATE를 반복해야 할까.
그렇지 않다.
DDL 문 끝에 `ON CLUSTER <클러스터명>`을 붙이면, 그 스키마 변경이 클러스터의 모든 노드에 전파된다(공식 문서 기준).

```sql
CREATE TABLE uk.uk_price_paid_local ON CLUSTER cluster_2S_1R
( ... )
ENGINE = MergeTree
ORDER BY ( ... );
```

이렇게 한 번만 실행하면 `cluster_2S_1R`에 속한 모든 서버에 같은 로컬 테이블이 생긴다.
CREATE DATABASE, CREATE TABLE 같은 변경이 모든 노드에 일괄 적용된다.

## 샤딩 키: 어느 샤드로 보낼지 정하는 규칙

이제 핵심이다.
INSERT 한 행이 들어왔을 때, ClickHouse는 그 행을 어느 샤드로 보낼지 어떻게 정할까.

공식 문서의 규칙은 이렇다.
"데이터 행이 보내질 샤드를 고르기 위해, 샤딩 식을 계산하고 그 값을 전체 샤드 가중치 합으로 나눈 나머지(remainder)를 취한다"(공식 문서 기준).
즉 `샤딩식의 값 mod 전체 가중치` 가 어느 샤드 구간에 떨어지느냐로 결정된다.

여기서 **가중치(weight)**로 샤드별 데이터 비율을 조절할 수 있다.
공식 예시: 샤드가 둘인데 첫 샤드의 가중치가 1, 둘째가 2라면, 첫 샤드는 삽입 행의 3분의 1(1/3)을, 둘째는 3분의 2(2/3)를 받는다(공식 문서 기준).
서버 사양이 다를 때(한쪽이 더 큰 디스크) 데이터를 비례 배분하는 데 쓴다.

샤딩 식으로는 무엇을 쓸 수 있나.

- `rand()` — 랜덤.
  `sharding_key`를 아예 지정하지 않으면 기본값이 `rand()`다.
- 컬럼명 그대로.
- 해시 함수 — `intHash64(UserID)`, `cityHash64(...)` 등.

데이터를 고르게 흩뿌리고 싶다면, 공개적으로 권장되는 방식은 고카디널리티(값 종류가 많은) 컬럼을 해시하는 것이다.
`cityHash64`는 Google CityHash 기반 64비트 결정적 해시로 샤딩에 흔히 쓰인다.

```sql
-- rand() 대신 고카디널리티 컬럼 해시로 샤딩 → 결정적·균등 분산
CREATE TABLE events.events_distributed
ENGINE = Distributed('cluster_2S_1R', 'events', 'events_local', cityHash64(user_id));
```

반대로 날짜·도시·이벤트타입처럼 값 종류가 적은(저카디널리티) 컬럼을 샤딩 키로 직접 쓰면, 특정 샤드에만 데이터가 몰리는 hot shard 문제가 생길 수 있다.
다만 이 hot shard 회피나 `cityHash64` 권장은 공식 docs가 아니라 서드파티 정리에 기반한 권고라는 점은 밝혀 둔다 — 원리는 타당하지만 1차 출처는 아니다.

`rand()`는 분산은 균등하지만 같은 키가 매번 다른 샤드로 갈 수 있어, 같은 사용자의 데이터를 한 샤드에 모으는(co-location) 효과는 없다.
해시 샤딩은 같은 `user_id`가 항상 같은 샤드로 가므로 결정적이다.

## 쓰기는 기본이 비동기다

INSERT가 어떻게 동작하는지가 운영에서 특히 중요하다.
Distributed 테이블에 INSERT하면, 기본 동작은 **비동기**다(공식 문서 기준).

공식 설명을 그대로 옮기면, 데이터 블록은 일단 initiator(쿼리를 받은 노드)의 "로컬 파일시스템에 그냥 기록"되고, 그 뒤 "가능한 한 빨리 백그라운드로 원격 서버들에 전송"된다.
전송 대기 중인 데이터는 테이블 디렉터리 안의 파일 목록으로 확인할 수 있다.
소스 코드 레벨에서는 `DistributedAsyncInsertDirectoryQueue`라는 컴포넌트가 이 큐를 관리한다(소스 코드 확인 기준).

비동기의 장점은 INSERT가 빠르게 반환된다는 것이고, 단점은 INSERT가 성공해도 그 데이터가 아직 원격 샤드에 안 도착했을 수 있다는 것이다.
방금 넣은 데이터를 즉시 모든 샤드에서 조회해야 한다면 동기 모드가 필요하다.

```sql
-- INSERT은 기본 비동기(로컬 FS 큐잉 후 백그라운드 전송).
-- 즉시 원격 반영이 필요하면 동기 모드:
INSERT INTO events.events_distributed SETTINGS distributed_foreground_insert = 1
VALUES (...);  -- 구버전 별칭: insert_distributed_sync = 1
```

`distributed_foreground_insert = 1`(구버전 별칭 `insert_distributed_sync = 1`)로 두면, INSERT가 모든 대상 원격 샤드에 쓰기가 완료될 때까지 동기로 기다린다.
소스 코드상 동기면 `writeSync`, 비동기면 `writeAsync` 경로로 갈린다(소스 코드 확인 기준).
동기 모드는 즉시 일관성을 주는 대신 네트워크·원격 쓰기 지연만큼 느리다.

참고로 이 별칭이 정확히 어느 버전부터 통합됐는지 버전 번호까지는 공개적으로 확인하지 못했으니, 쓰기 전 사용 중인 버전의 설정명을 확인하길 권한다.

또 하나의 디테일: 샤딩 키 없이 다중 샤드에 INSERT하면 예외가 발생한다(소스 코드 확인 기준).
어느 샤드로 보낼지 알 수 없기 때문이다.
단, `insert_distributed_one_random_shard = 1`을 켜면 예외 대신 랜덤한 단일 샤드로 보낸다.

## 백그라운드 전송을 제어하는 설정들

비동기 전송에는 여러 제어 노브가 있다(공식 문서 기준).
배치·주기를 조절하는 `distributed_background_insert_batch`, `distributed_background_insert_sleep_time_ms`, `distributed_background_insert_max_sleep_time_ms` 같은 설정이 있고,
디스크 동기화용 `fsync_after_insert`, `fsync_directories`,
백프레셔(밀린 데이터가 너무 쌓일 때 보호)용 `bytes_to_throw_insert`(임계 초과 시 예외)와 `bytes_to_delay_insert`(지연, `max_delay_to_insert` 기본 60초) 등이 있다.

여기서 한 가지 함정을 짚는다.
이 백그라운드 인서트 설정군은 과거에 `distributed_directory_monitor_*`라는 이름이었다가 `distributed_background_insert_*`로 이름이 바뀌었다.
그래서 오래된 블로그나 예제와 설정명이 다를 수 있다 — 구명을 만나면 신명으로 매핑해서 읽으면 된다.
다만 어느 버전에서 정확히 바뀌었는지, 완전한 매핑표는 이번 정리에서 버전별로 확정하지 못했다.

## 복제와의 관계: internal_replication

8장에서 다룰 복제(ReplicatedMergeTree)와 Distributed가 만나는 지점이 `internal_replication` 설정이다.
샤드 안에 레플리카가 여러 개일 때, 누가 복제를 책임지느냐의 문제다.

공식 문서 기준으로 정리하면 이렇다.

| internal_replication | 쓰기 동작 | 복제 책임 |
|---|---|---|
| `true` | Distributed가 샤드 내 **정상 레플리카 1개에만** 씀 | ReplicatedMergeTree가 레플리카 간 복제 담당 |
| `false` (기본) | Distributed가 샤드 내 **모든 레플리카에 직접** 씀 | Distributed가 복사본을 직접 뿌림 |

소스 코드상으로는 `writeAsyncImpl`이 `shard_info.hasInternalReplication()`로 분기한다(소스 코드 확인 기준).

권장은 명확하다.
복제 테이블(ReplicatedMergeTree)을 쓴다면 `internal_replication = true`로 둔다.
`false`(기본)는 Distributed가 모든 레플리카에 직접 쓰는데, 이 경우 레플리카 간 일관성 보장이 약하다.
한 레플리카에는 들어가고 다른 레플리카엔 전송이 실패하면 둘이 어긋날 수 있기 때문이다.
복제는 복제 엔진에게 맡기는 게 원칙이다.

## 읽기는 어떻게 모이나: 2단계 집계

이제 쿼리(SELECT) 차례다.
Distributed 테이블에 SELECT를 던지면 무슨 일이 일어날까.

쿼리를 받은 노드가 **initiator**가 된다.
initiator는 쿼리를 재작성한 뒤 모든 샤드로 팬아웃(fan-out)하고, 각 샤드가 보내온 부분 결과를 받아 로컬에서 병합한다(공식 문서 기준).
집계(aggregation)는 특히 두 단계로 나뉜다.

- **1단계(샤드)**: 각 샤드가 자기 로컬 데이터에 서브쿼리를 실행하고 부분 집계를 만든다.
- **2단계(initiator)**: initiator가 각 샤드의 부분 집계 상태를 받아 최종 병합한다.

`GROUP BY count()`를 예로 들면, 각 샤드가 자기 데이터의 부분 카운트를 내고, initiator가 그것들을 합쳐 최종 카운트를 만든다.
전체 데이터를 한 곳으로 끌어모으지 않고 "각자 부분 집계 → 모아서 최종 집계"로 처리하기 때문에 네트워크로 오가는 양이 줄어든다.

이 병합 단계는 `distributed_group_by_no_merge` 설정으로 제어할 수 있다.
공개적으로 확인되는 범위에선, 1이면 샤드가 끝까지 처리하고 initiator는 단순 프록시처럼 동작하며, 2면 initiator가 `ORDER BY`/`LIMIT` 정도만 적용한다.
다만 이 설정의 정확한 현행 동작은 PR과 서드파티 정리에 기반하므로, 정밀한 의미는 최신 설정 문서에서 다시 확인하는 게 안전하다.
일반적으로는 자동 2단계 병합이 기본이고, 이 설정은 최적화용 노브다.

## 분산 서브쿼리의 함정: IN vs GLOBAL IN

분산 환경에서 초보자가 가장 잘 밟는 지뢰가 서브쿼리다.
오른쪽에 또 다른 분산 테이블이 오는 IN 절을 보자.

일반 `IN`은 쿼리가 각 원격 서버로 보내져 각자 로컬에서 서브쿼리를 실행한다.
그런데 오른쪽 서브쿼리가 분산 테이블이면, 그 서브쿼리가 다시 모든 원격 서버로 재전송된다.
N대 클러스터에서 이게 N×N, 즉 N제곱의 요청으로 폭증할 수 있다 — 100대면 1만 요청(공식 문서 기준).

`GLOBAL IN`은 이걸 막는다.
initiator가 서브쿼리를 **딱 한 번만** 실행해 임시 테이블에 결과를 모은 뒤, 그 임시 테이블을 각 샤드에 보낸다.
그래서 요청 폭증이 일어나지 않는다.
이때 네트워크 전송량을 줄이려면 서브쿼리에 `DISTINCT`를 붙여 중복을 미리 제거하길 권한다.

```sql
-- 일반 IN: 오른쪽이 분산 테이블이면 서브쿼리가 샤드마다 재전송 → 요청 폭증 위험
SELECT count() FROM distributed_table
WHERE user_id IN (SELECT user_id FROM another_distributed_table WHERE active);

-- GLOBAL IN: initiator가 서브쿼리를 1회만 실행 → 임시 테이블을 각 샤드에 전송
SELECT count() FROM distributed_table
WHERE user_id GLOBAL IN (SELECT DISTINCT user_id FROM another_distributed_table WHERE active);
```

이 동작은 `distributed_product_mode` 설정에도 영향을 받는데, 그 값별(local/global/allow/deny) 구체 동작은 IN 연산자 문서에 명시되어 있지 않아 별도 설정 문서를 확인해야 한다.

## "왜 이렇게?" — 굳이 2단으로 나눈 이유

마지막으로 한 발 물러서서 묻자.
왜 ClickHouse는 데이터를 담는 로컬 테이블과, 그 위의 Distributed 테이블을 따로 두었을까.

답은 **관심사의 분리**다.
실제 저장과 머지·압축·인덱싱은 로컬 MergeTree가 책임진다 — 한 노드 안의 일이다.
샤드를 가로지르는 라우팅(어디로 쓸지, 어디서 읽어 모을지)은 Distributed가 책임진다.
이렇게 나누면 로컬 테이블은 1장부터 8장까지 배운 그대로 동작하고, 그 위에 분산 계층만 얹으면 된다.
한 노드용으로 잘 만든 엔진을 그대로 재활용하면서, 수평 확장은 별도 계층으로 처리하는 설계다.

그래서 운영 관점의 핵심을 다시 정리하면 이렇다.
샤딩 키는 고르게 흩뿌리도록 신중히 고르고(몰림 주의), 쓰기는 기본 비동기임을 이해하고(즉시 일관성이 필요하면 동기 모드), 복제와 섞일 땐 `internal_replication = true`로 복제 책임을 복제 엔진에 넘기고, 분산 서브쿼리에선 `GLOBAL IN`을 떠올린다.

3장에서 본 한 노드의 MergeTree가 이 장에서 클러스터로 확장되었다.
바로 다음, 10장에서는 이 분산 환경에서 삽입을 어떻게 효율적으로 다루는지(특히 async insert 패턴)를 깊게 들여다본다.
삽입이 잦고 작게 들어오는 워크로드에서 ClickHouse를 살리는 결정적 패턴이다.

## 참고

- ClickHouse Docs — Distributed Table Engine: https://clickhouse.com/docs/engines/table-engines/special/distributed
- ClickHouse Docs — Horizontal Scaling (cluster, ON CLUSTER, local + distributed tables): https://clickhouse.com/docs/architecture/horizontal-scaling
- ClickHouse Docs — IN Operators (IN vs GLOBAL IN): https://clickhouse.com/docs/sql-reference/operators/in
- ClickHouse Blog — Multi-stage distributed query execution: https://clickhouse.com/blog/multi-stage-distributed-query-execution-clickhouse-cloud
- ClickHouse 소스코드 분석 (DistributedSink / DistributedAsyncInsertDirectoryQueue / internal_replication 분기) — DeepWiki: https://deepwiki.com/search/how-does-the-distributed-table_18a938c5-0c72-4034-a3b9-d11aaf6c94bc
