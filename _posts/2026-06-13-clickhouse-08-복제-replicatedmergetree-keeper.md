---
title: "ClickHouse 완전 정복 8장 — 복제: ReplicatedMergeTree와 ClickHouse Keeper"
date: 2026-06-13 23:51:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "운영"]
---
지금까지는 서버 한 대 안에서의 이야기였다.
3장의 part, 4장의 sparse 인덱스, 5~7장의 스키마·뷰까지 — 전부 "한 노드가 어떻게 데이터를 빠르게 저장하고 읽나"였다.
그런데 그 노드 한 대가 죽으면?
디스크가 깨지면?

답이 **복제(replication)**다.
같은 데이터를 여러 서버에 똑같이 두고, 한 대가 죽어도 나머지가 계속 서비스하게 만드는 것.
ClickHouse에서 이걸 담당하는 게 **ReplicatedMergeTree** 엔진과, 그 뒤를 조율하는 **ClickHouse Keeper**다.

이 장은 두 개의 큰 질문에 답한다.

1. **ReplicatedMergeTree** — 복제는 무엇을 복제하고, 어떻게 일관성을 맞추나.
2. **Keeper** — 그 조율을 누가 하나, 왜 ZooKeeper가 아니라 Keeper인가.

그리고 그 사이에 끼어 있는 **블록 중복제거**, **insert_quorum**, **macros / ON CLUSTER** 같은 실전 톱니바퀴들을 본다.

## 1. 복제는 "테이블 단위"다 — 서버 통째가 아니다

먼저 가장 흔한 오해부터 깬다.
ClickHouse의 복제는 **서버 전체를 통째로 미러링하는 게 아니다.**
복제는 **개별 테이블 단위**로 동작하고, 각 샤드는 자기만의 독립적인 복제를 가진다(공식 문서 기준).

즉 한 서버 안에 **복제되는 테이블과 안 되는 테이블이 동시에** 있을 수 있다.
`events` 테이블만 ReplicatedMergeTree로 만들면 그 테이블만 복제되고, 같은 서버의 임시 `MergeTree` 테이블은 그 서버에만 머문다.

> 비유: 복제는 "건물 한 채를 통째로 복사"가 아니라 **"중요한 서류철만 골라 다른 지점에도 같은 사본을 둔다"**에 가깝다.
> 이건 이해를 돕는 비유다 — 실제로는 테이블의 데이터 파트가 레플리카 사이를 오간다(아래에서 본다).

## 2. 비동기 멀티마스터 — 그리고 최종 일관성

ReplicatedMergeTree의 복제 모델은 **비동기 멀티마스터(asynchronous multi-master)**다(공식 문서 기준).
이 한 단어에 핵심이 다 들어 있으니 풀어보자.

- **멀티마스터** — `INSERT`(그리고 `ALTER`)를 **아무 레플리카에나** 보낼 수 있다.
  "쓰기 전용 마스터 한 대"가 따로 없다.
  어느 서버로 보내든 받는다.
- **비동기** — 한 서버에 들어온 데이터는 **네트워크 지연만큼 시차를 두고** 나머지 레플리카로 퍼진다.
  쓰자마자 모든 노드에 동시에 박히는 게 아니다.

그래서 이 모델이 제공하는 건 **최종 일관성(eventual consistency)**이다.
레플리카 하나가 잠깐 다운돼 있어도 문제없다 — 다시 살아나면 그동안 밀린 변경을 따라잡는다(공식 문서 기준).

여기서 중요한 구분 하나.
**무엇이 복제되고 무엇이 안 되나?**

| 쿼리 | 복제되나? | 비고 |
|---|---|---|
| `INSERT` | O | 압축된 데이터 파트가 복제됨 |
| `ALTER` | O | 데이터 변경분이 복제됨 |
| `CREATE` / `DROP` | X | 단일 서버에서만 실행 |
| `ATTACH` / `DETACH` | X | 단일 서버에서만 실행 |
| `RENAME` | X | 단일 서버에서만 실행 |

즉 **데이터(INSERT/ALTER)는 자동으로 퍼지지만, 테이블 구조를 만들고 지우는 DDL은 안 퍼진다**(공식 문서 기준).
클러스터 전체에 테이블을 만들려면 따로 손을 써야 하는데, 그게 뒤에 나오는 `ON CLUSTER`다.

## 3. 조율은 누가 하나 — ClickHouse Keeper

레플리카들이 서로 "내가 지금 어디까지 받았다", "이 블록은 이미 들어왔다" 같은 걸 맞추려면 **공통의 합의 장소**가 필요하다.
그 역할을 하는 게 **ClickHouse Keeper**다.

복제에 필요한 메타정보 — 레플리카 정보, 복제 큐, 블록 해시 등 — 가 전부 Keeper에 저장된다(공식 문서 기준).
ZooKeeper 3.4.5+ 도 여전히 쓸 수 있지만, 공식적으로 **Keeper가 권장**된다.

> 핵심: **데이터 자체는 Keeper에 저장되지 않는다.** Keeper에 들어가는 건 "누가 무엇을 했는지"를 맞추는 **조율 메타데이터**뿐이다. 실제 데이터 파트는 레플리카 디스크에 있다.

![](/assets/img/posts/clickhouse-08-복제-replicatedmergetree-keeper/clickhouse-replication-keeper.svg)

### Keeper는 무엇인가 — Raft, 그리고 ZooKeeper 호환

Keeper는 **Raft 합의 알고리즘**을 쓴다.
구체적으로는 eBay가 만든 **NuRaft**라는 경량 C++ 라이브러리를 사용하고, ZooKeeper가 쓰던 ZAB 프로토콜을 Raft로 대체했다(공식 블로그 기준).

그러면서도 **ZooKeeper 와이어 프로토콜과 호환**된다.
덕분에 클라이언트 입장에선 Keeper가 그냥 표준 ZooKeeper 앙상블처럼 보인다 — 클라이언트 코드를 바꿀 필요가 없다(공식 블로그 기준).

> 비유: Keeper는 **"같은 콘센트 규격을 쓰는 더 효율적인 발전기"**다. 플러그(클라이언트)는 그대로 꽂으면 되고, 안에서 전기를 만드는 방식만 바뀌었다.

### 왜 C++로 다시 만들었나

ZooKeeper는 Java 생태계 프로젝트라 ClickHouse의 C++ 코드베이스에 잘 들어맞지 않았다(공식 블로그 기준).
스케일이 커지면서 리소스 효율이 중요해졌고, 그래서 밑바닥부터 C++로 재구현한 게 Keeper다.

## 4. Keeper가 ZooKeeper보다 권장되는 이유

공식 KB(지식베이스)가 드는 구체적 이유들을 표로 정리하면(공식 문서 기준):

| 측면 | Keeper의 이점 |
|---|---|
| 디스크 사용 | 스냅샷·로그를 더 잘 압축 → 디스크를 훨씬 적게 씀 |
| 크기 제한 | 패킷/노드 데이터의 **1MB 제한이 없음** |
| zxid 오버플로 | 없음 (ZooKeeper는 **20억 트랜잭션마다 강제 재시작**) |
| 장애 복구 | 네트워크 파티션 후 더 빠른 복구 |
| 메모리 | 메모리 효율이 더 좋음 |
| 운영 | JVM 힙·GC 튜닝 불필요 → 셋업이 간단 |
| 전용 명령 | ReplicatedMergeTree용 커스텀 명령으로 일부 연산 가속 |
| 검증 | Jepsen 테스트 커버리지가 더 넓음 |

특히 **zxid 오버플로**는 ZooKeeper 운영자라면 한 번쯤 데인 지점이다 — 트랜잭션 ID가 한계에 닿으면 강제 재시작이 걸린다.
Keeper엔 그 한계가 없다.

벤치마크 한 가지를 덧붙이면, 공식 블로그의 한 테스트에서 Keeper는 같은 데이터량 대비 ZooKeeper보다 **최대 46배 적은 메모리**를 쓰면서 성능은 ZooKeeper에 근접했다(2023-09 시점 공식 블로그 기준).
다만 이 "46배"는 특정 워크로드·시점의 측정값이라, 절대 수치로 일반화하기보다 "메모리 효율이 크게 개선됐다"는 방향성으로 읽는 게 맞다.

### 일관성 측면 한 가지

Keeper는 Raft 구현으로 **쓰기뿐 아니라 읽기에도 선형성(linearizability)**을 보장한다고 공식 자료가 설명한다.
공개적으로 알려진 범위에선 ZooKeeper는 읽기에 대해 선형성을 보장하지 않아 클라이언트가 오래된(stale) 값을 읽을 수 있다(이 점은 ZooKeeper 1차 문서로 별도 확인이 더 필요한 영역이다).
대신 선형 쓰기는 순서대로 한 번에 하나씩 처리돼야 해서, 멀티코어로 무한정 확장되지는 않는다는 제약이 따른다(공식 자료 기준).

### 노드는 몇 개, 어디에 두나

가용성과 쿼럼 유지를 위해 **최소 3개의 Keeper 노드**가 권장된다(공식 문서 기준).
Raft 합의 특성상 과반(쿼럼)이 살아 있어야 하므로, 3대면 1대가 죽어도 나머지 2대로 운영이 이어진다.

그리고 프로덕션에서는 **Keeper를 ClickHouse Server와 같은 호스트가 아닌 전용 호스트**에서 돌리는 것이 강력히 권장된다(공식 문서 기준).
무거운 쿼리가 Keeper의 응답을 방해하지 않게 하려는 분리다.

## 5. 블록 중복제거 — 재시도해도 안전한 이유

비동기 멀티마스터에는 골치 아픈 문제가 하나 따라온다.
**같은 데이터를 두 번 보내면?**

네트워크가 끊겨 `INSERT`가 성공했는지 애매할 때, 보통은 재시도한다.
그런데 사실 첫 시도가 성공했다면 — 데이터가 두 번 들어가버린다.
ReplicatedMergeTree는 이걸 **블록 중복제거(deduplication)**로 막는다(공식 문서·소스 기준).

작동 방식은 이렇다.

1. `INSERT`된 각 데이터 블록에 **해시**를 계산한다.
2. 그 해시를 **Keeper에 저장**해 둔다.
3. 새 파트를 커밋하기 전, **같은 해시가 이미 있는지 검사**한다.
4. 이미 있으면 → 그 블록은 **기록하지 않고 건너뛴다.**

그래서 같은 블록을 여러 번 보내거나, 심지어 **다른 레플리카로 다시 보내도 한 번만 기록**된다.
이 성질이 "재시도해도 안전(idempotent)"을 만든다 — 멀티마스터 환경에서 마음 놓고 재시도할 수 있는 근거다.

이 중복제거의 범위는 `replicated_deduplication_window` 설정으로 제어된다(소스 기준).
이건 **Keeper가 해시를 보관할 "최근 삽입 블록 개수"**다.
시간 기반인 `replicated_deduplication_window_seconds`도 있고, async insert(10장)용 별도 설정도 따로 있다.
윈도우를 벗어난 오래된 해시는 백그라운드 정리 스레드가 치운다.

> 주의: 이 설정들의 정확한 기본값은 버전에 따라 다를 수 있어, 실제 운영 전 사용 중인 버전의 설정 문서로 확인하는 게 안전하다.

## 6. insert_quorum — 데이터 유실을 막는 안전벨트

기본 동작에서 `INSERT`는 **단 하나의 레플리카**가 확인하면 성공으로 반환된다(`insert_quorum = 0`, 비활성).
빠르지만, 만약 그 레플리카가 데이터를 다른 곳에 퍼뜨리기 전에 죽으면 — 그 쓰기는 사라질 수 있다.

이걸 막는 게 **insert_quorum**이다(공식 문서·소스 기준).
`insert_quorum = N` (N>0) 으로 두면, **N개 레플리카가 쓰기를 확인해야** 비로소 `INSERT`가 성공으로 반환된다.

```sql
-- 2개 레플리카 확인을 받아야 INSERT 성공 (유실 방지)
INSERT INTO events SETTINGS insert_quorum = 2 VALUES (today(), 1, 100);
```

쿼럼이 제한 시간 안에 채워지지 않으면 예외가 던져진다 — "조용히 유실"이 아니라 "명시적 실패"로 바뀌는 것이다.

### 쿼럼 쓰기와 읽기 일관성의 미묘함

여기 함정이 하나 있다.
`insert_quorum_parallel`이라는 설정이 기본 활성(1) 상태인데, 이건 같은 테이블에 대한 **병렬 쿼럼 INSERT를 허용**해 처리량을 높인다(공개 자료 기준).
대신 이 상태에서는 **읽기 순차 일관성(`select_sequential_consistency`)이 효과가 없다.**

그래서 "쿼럼으로 쓴 직후 그 값을 반드시 최신으로 읽고 싶다"면, 세 가지를 **함께** 설정해야 한다.

| 설정 | 값 | 의미 |
|---|---|---|
| `insert_quorum` | > 0 | N개 레플리카 확인 후 쓰기 성공 |
| `insert_quorum_parallel` | 0 | 병렬 쿼럼 끔 (순차 일관성 활성화 조건) |
| `select_sequential_consistency` | 1 | 항상 최신 데이터를 읽음 |

```sql
-- 쿼럼 쓰기 직후 항상 최신을 읽으려면 세 설정을 함께:
--   insert_quorum > 0, insert_quorum_parallel = 0, select_sequential_consistency = 1
SELECT count() FROM events
SETTINGS select_sequential_consistency = 1;
```

이건 명백한 trade-off다 — **처리량(병렬 쿼럼)** vs **읽기 일관성(순차 보장)**.
대부분의 분석 워크로드는 최종 일관성으로 충분하므로, 이 강한 일관성 조합은 정말 필요한 경우에만 켠다.
(설정 간 상호작용과 기본값은 버전에 따라 달라질 수 있어, 적용 전 현행 문서 확인을 권한다.)

## 7. macros와 ON CLUSTER — 여러 노드를 한 번에

이제 실전에서 테이블을 어떻게 만드는지 본다.

ReplicatedMergeTree는 두 개의 인자를 받는다 — **Keeper 경로**와 **레플리카 이름**이다.
그런데 노드가 여러 대인데 매번 경로와 이름을 손으로 다르게 적으면 실수가 난다.
그래서 **macros(매크로)**를 쓴다.

`{shard}`와 `{replica}` 같은 매크로를 ENGINE 정의에 쓰고, 실제 치환값은 **각 서버의 설정 파일**에서 노드별로 다르게 정의한다(공식 문서 기준).

```sql
-- 각 노드 config의 <macros>에서 {shard}/{replica} 치환값 정의
-- ENGINE 인자1: Keeper 경로(샤드별 고유), 인자2: 레플리카 이름
CREATE TABLE events ON CLUSTER my_cluster
(
    event_date Date,
    event_id   UInt64,
    user_id    UInt64
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/events',
    '{replica}'
)
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, event_id);
```

설정 파일 쪽은 노드마다 이렇게 다르게 둔다.

```sql
<!-- shard-1, replica-a 노드 예시 -->
<macros>
    <shard>01</shard>
    <replica>replica-a</replica>
</macros>
<!-- 같은 샤드의 다른 레플리카 노드는 <replica>replica-b</replica> -->
```

이렇게 하면 **모든 노드에 똑같은 CREATE 문**을 보내도, 각 노드가 자기 매크로로 경로·이름을 알아서 치환한다.
`{database}`/`{table}` 같은 내장 매크로도 있지만, `RENAME` 시 경로가 꼬일 수 있어 주의가 필요하다.

### ON CLUSTER — DDL을 클러스터 전체에

앞서 "CREATE/DROP 같은 DDL은 복제되지 않는다"고 했다.
그 한계를 메우는 게 `ON CLUSTER` 절이다.

`ON CLUSTER my_cluster`를 붙이면, 그 DDL이 단일 서버가 아니라 **클러스터에 정의된 모든 호스트**에서 실행된다(공식 문서 기준).
동작 조건은 두 가지 — 모든 호스트가 **동일한 cluster 정의**를 갖고 있어야 하고, **Keeper에 연결**돼 있어야 한다.

그리고 좋은 점 하나.
일부 호스트가 그 순간 다운돼 있어도, **distributed DDL 큐**를 통해 결국 각 호스트에서 실행된다 — 여기서도 최종 일관성이 작동한다(공식 문서 기준).

## 8. 복제의 비용 — 그리고 비용이 없는 곳

복제가 공짜는 아니다.
공식 문서에 따르면 `INSERT` 트랜잭션당 약 **10개의 Keeper(ZooKeeper) 엔트리**가 추가된다.
그래서 비복제 테이블 대비 **약간의 쓰기 레이턴시 증가**가 생긴다.

하지만 결정적으로 — **`SELECT`는 Keeper를 전혀 쓰지 않는다**(공식 문서 기준).
읽기 경로엔 조율이 끼지 않으니, 복제 테이블의 **읽기 성능은 비복제 테이블과 동등**하다.

| 작업 | Keeper 사용 | 영향 |
|---|---|---|
| `INSERT` | O (≈10 엔트리/트랜잭션) | 약간의 쓰기 레이턴시 증가 |
| `SELECT` | X | 비복제와 동등 |

이게 ClickHouse 설계의 합리적인 부분이다 — 조율 비용을 **쓰기에만** 몰고, **읽기는 깨끗하게** 비워둔다.

## 9. 운영 점검 — system 테이블로 복제 상태 보기

복제는 비동기라, "지금 레플리카들이 얼마나 벌어져 있나"를 들여다볼 창이 필요하다.
그게 `system.replicas`와 `system.replication_queue`다(13장에서 더 깊이 다룬다).

```sql
-- 레플리카 헬스: 지연·읽기전용·큐 길이
SELECT database, table, is_leader, is_readonly,
       absolute_delay, queue_size, inserts_in_queue, merges_in_queue
FROM system.replicas
WHERE table = 'events';

-- 복제 큐에 쌓인 개별 작업(머지/페치/뮤테이션)
SELECT type, create_time, num_tries, last_exception
FROM system.replication_queue
WHERE table = 'events'
ORDER BY create_time;
```

여기서 `absolute_delay`가 커지거나 `queue_size`가 계속 쌓이면, 복제가 따라잡지 못하고 있다는 신호다.
`is_readonly`가 1이면 보통 Keeper 연결이 끊긴 상태라, 그 레플리카는 쓰기를 못 받는다.

## 10. 복제와 샤딩은 다른 축이다 (9장 예고)

마지막으로 자주 헷갈리는 경계를 하나 정리한다.

- **복제(이 장)** — 같은 데이터를 **여러 벌** 둔다.
  목적은 **가용성·내구성**.
  같은 샤드의 레플리카끼리.
- **샤딩(9장)** — 데이터를 **쪼개서 나눠** 둔다.
  목적은 **용량·처리량 확장**.
  서로 다른 샤드끼리.

이 둘은 직교한다 — 보통 "샤드 여러 개 × 각 샤드마다 레플리카 여러 개"로 함께 쓴다.

그 둘을 잇는 설정이 **internal_replication**이다.
샤드 단위로 `internal_replication=true`로 두면, Distributed 테이블(9장)이 한 샤드의 **한 레플리카에만** 데이터를 쓰고, 나머지 레플리카로는 ReplicatedMergeTree가 **자체 복제**한다(공식 문서 기준).
이렇게 하면 쓰기가 중복되지 않는다.
`false`면 Distributed가 모든 레플리카에 직접 쓴다.

이 `internal_replication`과 Distributed 엔진이 바로 다음 9장의 주제다.

## 한 장으로 정리

| 개념 | 한 줄 |
|---|---|
| **ReplicatedMergeTree** | 테이블 단위·샤드별 독립의 비동기 멀티마스터 복제. 최종 일관성. |
| 복제 대상 | INSERT/ALTER의 데이터는 복제. CREATE/DROP 등 DDL은 복제 안 됨. |
| **Keeper** | 복제 조율 메타데이터 저장소. Raft(NuRaft) 기반, ZooKeeper 와이어 호환, C++. |
| Keeper > ZooKeeper | 디스크·메모리 효율, 1MB 제한·zxid 오버플로 없음, 운영 간단. |
| 블록 중복제거 | 블록 해시를 Keeper에 저장해 재시도/중복 INSERT를 한 번만 기록. |
| **insert_quorum** | N개 레플리카 확인 후 쓰기 성공 → 유실 방지. 강한 일관성은 별도 설정 조합. |
| macros / ON CLUSTER | `{shard}`/`{replica}` 치환 + DDL을 클러스터 전체에 실행. |
| 비용 | INSERT는 ≈10 Keeper 엔트리. SELECT는 Keeper 미사용 → 비복제와 동등. |

복제로 "한 대가 죽어도 데이터가 남는" 안전망을 만들었다.
하지만 데이터가 한 대에 다 안 들어갈 만큼 커지면?
그때는 데이터를 **쪼개서 여러 샤드에** 나눠 담아야 한다.
그 분산·샤딩과 Distributed 테이블 — 9장에서 본다.

---

## 참고

- ClickHouse 공식 문서 — Data Replication: https://clickhouse.com/docs/engines/table-engines/mergetree-family/replication
- ClickHouse 공식 문서 — Replication architecture: https://clickhouse.com/docs/architecture/replication
- ClickHouse 공식 문서 — Distributed DDL (ON CLUSTER): https://clickhouse.com/docs/sql-reference/distributed-ddl
- ClickHouse 공식 KB — Why recommend ClickHouse Keeper over ZooKeeper: https://clickhouse.com/docs/knowledgebase/why_recommend_clickhouse_keeper_over_zookeeper
- ClickHouse 공식 KB — Read consistency: https://clickhouse.com/docs/knowledgebase/read_consistency
- ClickHouse 공식 블로그 — ClickHouse Keeper: a ZooKeeper alternative written in C++: https://clickhouse.com/blog/clickhouse-keeper-a-zookeeper-alternative-written-in-cpp
- ClickHouse/ClickHouse 소스 (블록 중복제거·insert_quorum 동작) — DeepWiki: https://deepwiki.com/ClickHouse/ClickHouse
