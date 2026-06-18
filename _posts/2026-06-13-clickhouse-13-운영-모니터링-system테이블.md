---
title: "ClickHouse 완전 정복 13장 — 운영·모니터링: system 테이블로 서버 속을 들여다보기"
date: 2026-06-13 23:46:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "운영"]
---
지금까지 우리는 ClickHouse가 *어떻게 빠른지*, 데이터가 어떻게 저장되고 머지되고 복제되는지를 봤다.
이번 장은 시점이 다르다.
"이미 돌아가고 있는 서버가 지금 건강한가?" 를 물어본다.

그리고 ClickHouse는 이 질문에 답하기 위한 도구를 아주 ClickHouse답게 제공한다.
바로 **자기 자신을 테이블로 보여주는 것**이다.

## 첫 원리 — 서버 상태가 곧 테이블이다

보통의 데이터베이스에서 "서버 상태를 본다"는 건 로그 파일을 뒤지거나, 전용 관리 도구를 켜거나, 별도 명령어를 외우는 일이다.
ClickHouse는 그걸 자기가 제일 잘하는 방식으로 바꿔버렸다.
"우리는 SQL로 데이터를 분석하는 데 특화돼 있다.
그러면 서버의 내부 상태도 그냥 테이블로 노출해서 SQL로 분석하게 하면 되지 않나?"

그 결과가 `system` 데이터베이스다.
지금 어떤 파트가 디스크에 있는지, 어떤 머지가 진행 중인지, 방금 어떤 쿼리가 느렸는지, 복제가 몇 초나 뒤처졌는지 — 이 모든 게 `system.*` 테이블의 행으로 들어온다.
운영자는 새 언어를 배울 필요가 없다.
이미 아는 `SELECT ... WHERE ... GROUP BY` 로 서버를 진단한다.

비유를 하나 들자.
**system 테이블은 병원의 모니터링 차트다.**
환자(서버)의 심박·혈압·체온이 실시간으로 화면에 숫자로 뜨고, 의사(운영자)는 그 숫자를 읽고 이상을 판단한다.
다만 이건 어디까지나 이해를 돕는 비유다.
실제로 system 테이블은 "차트"라기보다 서버가 메모리에 들고 있는 카운터·상태를 그때그때 테이블 인터페이스로 비춰주는 뷰에 가깝고, 그 구분(순간값이냐 누적값이냐 주기 계산값이냐)이 운영에서 꽤 중요하다.
그 차이는 잠시 뒤에 짚는다.

## 8개의 핵심 system 테이블

운영에서 자주 보는 테이블은 의외로 몇 개 안 된다.
역할로 묶어보면 이렇다.

| 테이블 | 무엇을 보여주나 | 언제 보나 |
|---|---|---|
| `system.parts` | MergeTree 테이블의 모든 데이터 파트(행 단위) | too-many-parts 진단, 파티션별 파트 수, 압축률 |
| `system.merges` | 진행 중인 머지·뮤테이션 | 머지 적체, 오래 걸리는 머지 추적 |
| `system.mutations` | ALTER UPDATE/DELETE 진행 상황 | 뮤테이션 백로그·실패 점검 |
| `system.query_log` | 실행된 쿼리 단위 로그 | 느린 쿼리·메모리 다소비 쿼리 진단 |
| `system.metrics` | 지금 이 순간의 측정치(순간값) | 현재 실행 쿼리 수, 진행 머지 수 등 |
| `system.asynchronous_metrics` | 주기적으로 계산되는 지표 | 하드웨어 상태, 복제 지연, 머지 큐 |
| `system.replicas` | ReplicatedMergeTree 복제 상태 | 복제 지연·읽기전용·큐 적체 |
| `system.errors` | 서버 시작 이후 누적 에러 카운터 | 최근 에러 패턴 파악 |

하나씩 들여다보자.

### system.parts — 파트의 호적등본

3장에서 봤듯 ClickHouse는 INSERT마다 불변의 데이터 파트(컬럼 파일들이 든 디렉터리)를 만들고, 백그라운드 머지로 점점 큰 파트로 합친다.
`system.parts`는 이 파트 하나하나를 행으로 나열한다.

여기서 핵심은 `active` 컬럼이다.
머지가 끝나면 원본 작은 파트들은 "비활성"으로 표시되고 결과 큰 파트만 `active=1`이 된다(아직 삭제 전 상태로 잠시 남는다).
그래서 "지금 살아있는 파트가 몇 개냐"를 셀 땐 반드시 `WHERE active`를 건다.

```sql
SELECT
    database, table, partition,
    count() AS parts,
    sum(rows) AS rows,
    formatReadableSize(sum(bytes_on_disk)) AS size
FROM system.parts
WHERE active
GROUP BY database, table, partition
ORDER BY parts DESC
LIMIT 20;
```

이 쿼리 하나로 "어느 테이블의 어느 파티션에 파트가 비정상적으로 많이 쌓였는가"가 한눈에 보인다.
뒤에서 다룰 too-many-parts 진단의 출발점이 바로 이 숫자다.

### system.merges — 지금 합쳐지고 있는 것들

머지가 적체되면 작은 파트가 줄지 않고 계속 쌓인다.
`system.merges`는 *지금 진행 중인* 머지와 뮤테이션을 보여준다.
주요 컬럼은 `progress`(0~1 완료율), `elapsed`(경과 초), `num_parts`(병합 중인 파트 수), `is_mutation`(1이면 뮤테이션), `memory_usage`, `result_part_name` 등이다.

```sql
SELECT
    database, table,
    round(progress, 3) AS progress,
    elapsed,
    num_parts,
    is_mutation,
    formatReadableSize(memory_usage) AS mem,
    result_part_name
FROM system.merges
ORDER BY elapsed DESC;
```

`elapsed`가 큰데 `progress`가 잘 안 오르면, 그 머지가 막혀 있다는 신호다.

### system.mutations — UPDATE/DELETE의 진행 추적

1장에서 ClickHouse의 UPDATE/DELETE는 즉시 반영되는 게 아니라 **뮤테이션(mutation)** 이라는 비동기 작업으로 처리된다고 했다.
`system.mutations`는 그 뮤테이션의 진행을 추적한다.
`is_done`(1=완료), `parts_to_do`(처리 남은 파트 수), `latest_fail_reason`(실패 사유), `latest_fail_time` 같은 컬럼이 핵심이다.

```sql
SELECT
    database, table, mutation_id, command,
    parts_to_do, is_done,
    latest_fail_time, latest_fail_reason
FROM system.mutations
WHERE is_done = 0
ORDER BY create_time ASC;
```

`is_done=0`인 채로 진행이 멈춰 있거나 `latest_fail_reason`이 채워져 있으면 적체·실패다.
단, 한 가지 함정이 있다.
`parts_to_do=0`이어도 뮤테이션이 아직 미완료로 남을 수 있다 — ReplicatedMergeTree에서 장기 실행 INSERT가 끝나길 기다리는 경우다(공식 문서에 명시).
그러니 `parts_to_do`만 보고 "끝났다"고 단정하지 말고 `is_done`을 같이 봐야 한다.

### system.query_log — 느린 쿼리의 1차 수사 도구

`system.query_log`는 실행된 쿼리를 한 줄씩 남긴다.
한 쿼리에 대해 `type`이 `QueryStart`, `QueryFinish`, 그리고 예외 시 `ExceptionBeforeStart`/`ExceptionWhileProcessing` 등으로 기록된다.
운영에서 가장 자주 쓰는 컬럼은 `query_duration_ms`(쿼리 소요 시간), `memory_usage`/`peak_memory_usage`(메모리 사용·피크), `read_rows`/`read_bytes`(읽은 행·바이트), `user`, `event_time`, `query`(쿼리 원문)다.

느린 쿼리 추적은 이렇게 한다.

```sql
SELECT
    event_time, user,
    query_duration_ms,
    formatReadableSize(read_bytes) AS read,
    formatReadableSize(memory_usage) AS mem,
    query
FROM system.query_log
WHERE type = 'QueryFinish'
  AND is_initial_query
  AND event_time > now() - INTERVAL 1 DAY
ORDER BY query_duration_ms DESC
LIMIT 10;
```

이게 "지난 하루 가장 느렸던 쿼리 Top 10"이다.
메모리를 많이 먹는 쿼리를 찾고 싶으면 정렬 기준을 `peak_memory_usage`로 바꾸면 된다.

query_log는 양이 많아질 수 있다.
공개적으로 권장되는 운영 팁으로는, `log_queries_min_query_duration_ms` 설정으로 일정 시간 미만의 쿼리는 아예 로깅에서 빼 노이즈를 줄이고, query_log 테이블에 TTL을 걸어 보존기간을 관리하는 방식이 있다.
다만 구체적 설정값·보존 기간은 환경에 따라 다르니 공식 settings 문서로 확인하는 게 좋다.

### system.metrics vs system.asynchronous_metrics — '순간값'과 '주기 계산값'

이 두 테이블은 이름이 비슷해서 헷갈리지만 성격이 다르다.
**핵심 차이는 "값이 언제 계산되느냐"다.**

`system.metrics`는 **즉시 계산되거나 지금 현재 값을 갖는 순간 측정치**다.
컬럼은 `metric`/`value`/`description` 세 개뿐이다.
대표적인 값:

| metric | 의미 |
|---|---|
| `Query` | 지금 실행 중인 쿼리 수 |
| `Merge` | 지금 진행 중인 머지 수 |
| `PartMutation` | 지금 진행 중인 뮤테이션 수 |
| `ReadonlyReplica` | 읽기전용 상태인 복제 테이블 수 |
| `BackgroundMergesAndMutationsPoolTask` | 머지/뮤테이션 풀의 활성 태스크 수 |
| `MemoryTracking` | 서버가 할당한 메모리(바이트) |
| `TCPConnection` | 현재 TCP 연결 수 |

```sql
SELECT * FROM system.metrics
WHERE metric IN (
    'Query', 'Merge', 'PartMutation',
    'ReadonlyReplica', 'BackgroundMergesAndMutationsPoolTask'
);
```

반면 `system.asynchronous_metrics`는 **백그라운드에서 주기적으로 계산되는 지표**다.
주로 하드웨어 상태나 집계가 필요한 값이 여기 들어온다.
`MemoryResident`(실제 점유 메모리), `OSMemoryAvailable`, `LoadAverage1`, `DiskAvailable_*`, `TotalBytesOfMergeTreeTables`, `Uptime`, 그리고 복제 관련으로 `ReplicasMaxAbsoluteDelay`, `ReplicasSumMergesInQueue` 같은 값이 있다.

```sql
SELECT metric, value
FROM system.asynchronous_metrics
WHERE metric IN (
    'ReplicasMaxAbsoluteDelay',
    'ReplicasSumMergesInQueue',
    'Uptime'
);
```

왜 이렇게 둘로 나눴을까?
"현재 실행 중인 쿼리 수" 같은 건 물어볼 때마다 즉시 셀 수 있다 — 순간값(`metrics`)이 어울린다.
반면 "디스크 남은 용량"이나 "MergeTree 테이블 총 바이트" 같은 건 매번 계산하면 비싸니, 일정 주기로 백그라운드에서 계산해 캐시해 두는 게 합리적이다 — 그래서 `asynchronous_metrics`다.
모니터링 대시보드를 짤 때 이 구분을 알아야 "내가 보는 숫자가 실시간인가, 몇 초 전 스냅샷인가"를 안다.

### system.replicas — 복제 건강검진

8장에서 본 ReplicatedMergeTree의 상태는 `system.replicas`에 담긴다.
복제 클러스터 운영에서 매일 봐야 하는 테이블이다.
핵심 컬럼:

| 컬럼 | 의미 |
|---|---|
| `is_readonly` | 읽기전용 — Keeper 연결 문제 등으로 쓰기 불가 상태 |
| `is_session_expired` | Keeper 세션이 끊김 |
| `absolute_delay` | 현재 복제 지연(초) |
| `queue_size` | 대기 중인 작업 총수 |
| `inserts_in_queue` / `merges_in_queue` | 큐에 쌓인 삽입·머지 작업 수 |
| `future_parts` | 곧 나타날 예정인 파트 수 |
| `parts_to_check` | 손상 의심으로 검사 대기 중인 파트 |
| `log_max_index` − `log_pointer` | 차이가 크면 동기화가 뒤처진 것 |
| `active_replicas` / `total_replicas` | 활성/전체 복제본 수 |

```sql
SELECT
    database, table,
    is_readonly, is_session_expired,
    absolute_delay, queue_size,
    inserts_in_queue, merges_in_queue,
    log_max_index - log_pointer AS log_lag,
    active_replicas, total_replicas
FROM system.replicas
WHERE is_readonly OR absolute_delay > 60 OR queue_size > 0;
```

`is_readonly`가 켜졌거나, `absolute_delay`가 수십 초 이상이거나, `queue_size`가 줄지 않으면 복제에 문제가 생긴 것이다.
`active_replicas`가 `total_replicas`보다 작으면 어떤 복제본이 죽었다는 뜻이다.

### system.errors — 누적 에러 카운터

`system.errors`는 서버 시작 이후 발생한 에러를 종류별로 카운트한다.
컬럼은 `name`(에러명), `code`(에러 코드), `value`(발생 횟수), `last_error_time`, `last_error_message`, `last_error_trace`, `remote`(분산 쿼리의 원격 노드에서 발생했는지) 등이다.

```sql
SELECT name, code, value, last_error_time, last_error_message
FROM system.errors
WHERE value > 0
ORDER BY last_error_time DESC
LIMIT 10;
```

다만 주의가 있다.
공식 문서에 따르면 성공한 쿼리 중에도 내부적으로 일부 에러 카운터가 증가할 수 있다.
즉 `system.errors`의 카운트가 곧바로 "장애"를 뜻하진 않으며, false positive가 있을 수 있다.
이 테이블은 단독 알람 지표보다는 "최근 어떤 에러들이 늘고 있나"를 패턴으로 보는 용도가 적합하다.

## 운영의 단골 함정 — 'Too many parts'

system 테이블을 왜 보는지 가장 잘 보여주는 사례가 too-many-parts다.

증상은 명확하다.
어느 순간 INSERT가 `TOO_MANY_PARTS`(에러 코드 252)로 거부된다.
이건 한 파티션의 활성 파트 수가 한도(`parts_to_throw_insert`)를 넘었다는 뜻이다.
그 한도에 닿기 전 단계인 `parts_to_delay_insert`를 넘으면, 먼저 INSERT에 인위적인 지연이 걸린다 — 일종의 경고 브레이크다.

현재 MergeTree 기본값은 다음과 같다(공식 문서·소스 기준).

| 설정 | 기본값 | 의미 |
|---|---|---|
| `parts_to_delay_insert` | 1000 | 파티션 활성 파트가 이를 넘으면 INSERT에 지연 |
| `parts_to_throw_insert` | 3000 | 이를 넘으면 INSERT 거부(코드 252) |
| `max_parts_in_total` | 100000 | 테이블 전체 활성 파트 한도 |

참고로 23.6 이전에는 delay/throw 기본값이 150/300으로 훨씬 낮았는데, 23.6부터 1000/3000으로 상향됐다.
그래서 옛 자료에서 "300이 한도"라고 본 적이 있다면, 현행 버전과 다르다는 점을 기억하자.

여기서 가장 중요한 운영 원칙은 이것이다.
**too-many-parts는 거의 항상 설정 문제가 아니라 '쓰기 패턴' 문제다.**
한도를 올려서 막는 건 임시방편이고, 근본 원인은 보통 "작은 INSERT를 너무 자주 날린다"는 데 있다.

진단은 단순하다.

```sql
SELECT count() FROM system.parts
WHERE table = '<테이블명>' AND active;
```

이 숫자가 수천에 근접하면 위험 신호다.

### 왜 작은 INSERT가 문제일까?

3장의 원리로 돌아가 보자.
INSERT 한 번 = 새 파트 하나.
초당 수백 번씩 작은 INSERT를 날리면 작은 파트가 폭증하고, 백그라운드 머지가 그걸 따라잡지 못한다.
머지가 인입 속도를 못 따라가면 파트는 계속 쌓이고, 결국 한도를 넘어 INSERT가 막힌다.
실제로 OpenTelemetry 같은 데이터를 잘게 흘려넣다가 too-many-parts 장애를 겪은 운영 포스트모템 사례도 공개돼 있다.

공개적으로 권장되는 해법은 한도 조정이 아니라 쓰기 패턴 개선이다.

1. **배치 INSERT** — 작은 INSERT를 모아서 큰 덩어리로.
   공개 가이드는 대략 1~2초당 1회, 회당 1만~50만 행 수준의 템포를 권한다.
2. **async insert** — 클라이언트가 배치를 못 만드는 상황이면, 서버가 작은 INSERT를 버퍼에 모았다가 한꺼번에 flush하게 한다.
3. **파티션 키를 과도하게 잘게 쪼개지 않기** — 파티션이 많을수록 파트도 분산돼 머지가 불리해진다.
4. 필요하면 Buffer 테이블로 인입을 평탄화한다.

### async insert의 동작

async insert는 서버 버퍼에 INSERT를 모았다가 세 조건 중 **먼저 충족되는 것**에서 flush한다.
공개된 설명 기준 기본값은 다음과 같으나, 버전에 따라 달라질 수 있으니 정확한 현행값은 공식 settings 문서로 확인하는 게 좋다.

| 조건 | 기본값(공개 자료 기준) | 의미 |
|---|---|---|
| `async_insert_max_data_size` | 약 1MB | 버퍼 데이터가 이만큼 차면 flush |
| `async_insert_busy_timeout_ms` | 약 1초 | 마지막 flush 후 이 시간 지나면 flush |
| `async_insert_max_query_number` | 약 100 | 모인 INSERT 쿼리 수가 이만큼이면 flush |

또 `wait_for_async_insert=1`(기본)이면, 클라이언트는 데이터가 실제로 디스크에 기록된 뒤 ACK를 받는다.
덕분에 flush 전에 서버가 죽어도 데이터 유실을 막을 수 있다.
이 패턴의 더 자세한 인입 전략은 10장(삽입 패턴)에서 다뤘다.

## 머지 적체와 Keeper 부하 — 보이지 않는 연쇄

too-many-parts의 *뒤편*에는 종종 머지 적체가 있다.
공개된 운영 경험에 따르면, 늦게 도착한 데이터가 작은 파트를 만들면 그 머지가 수 시간에서 수일까지 걸릴 수 있고, 그동안 작은 파트가 더 쌓이는 악순환이 생긴다.
또 `ALTER UPDATE/DELETE` 뮤테이션은 머지와 같은 풀의 스레드를 점유하므로, 큰 테이블에 광범위한 뮤테이션을 걸면 그 기간 동안 정상 머지가 밀릴 수 있다.
이 부분은 1차 공식 문서라기보단 운영 경험 기반 자료가 많으므로, 메커니즘으로 이해하되 구체 수치는 환경마다 다르다고 보는 게 안전하다.

복제 환경이면 한 겹이 더 있다.
ReplicatedMergeTree에서 머지는 ClickHouse Keeper(8장)로 조율된다.
그래서 복제본이 뒤처지거나 Keeper가 과부하면, 머지 태스크가 `system.replication_queue`에 쌓인 채 실행되지 못하고 파트가 누적된다.
나쁜 파티셔닝 키나 과도한 작업이 Keeper에 부담을 주면 이 적체가 더 심해진다는 운영 사례도 보고된다(다만 구체 수치는 개인 운영 사례 기반이라 단정하긴 어렵다).
요점은 — **복제 클러스터에서 파트가 안 줄면, system.replicas와 복제 큐를 같이 봐야 한다**는 것이다.

## 메모리 한도 — 'Memory limit exceeded'

운영하다 보면 만나는 또 다른 단골이 `MEMORY_LIMIT_EXCEEDED`(에러 코드 241)다.
공개적으로 확인되는 범위에선, ClickHouse는 쿼리·유저·서버 단위로 **MemoryTracker** 계층을 두고 모든 메모리 할당을 추적하며, 가장 먼저 한도에 닿은 트래커가 해당 쿼리를 코드 241로 종료한다.
에러 메시지에 "(total)"이 붙은 'Memory limit (total) exceeded'는 서버 전역 한도(`max_server_memory_usage`)를 넘었다는 뜻이다.

진단 흐름은 system 테이블 안에서 끝난다.

- `system.query_log`에서 `peak_memory_usage` 기준 내림차순으로, 어떤 쿼리가 메모리를 많이 먹는지 찾는다.
- `system.metrics`의 `MemoryTracking`으로 지금 서버가 들고 있는 메모리를 확인한다.

다만 `max_server_memory_usage`의 도입 버전 등 일부 세부는 서드파티 자료에 기댄 부분이 있어, 정확한 동작은 공식 settings 문서로 한 번 더 확인하길 권한다.

## 밖으로 내보내기 — Prometheus와 Cloud 대시보드

지금까지는 사람이 직접 SQL을 날리는 방식이었다.
하지만 운영은 24시간 자동 감시가 필요하다.
ClickHouse는 내부 지표를 외부 모니터링 시스템으로 내보낼 수 있다.

먼저 분류를 정리하면, 내장 지표는 세 부류다(공식 monitoring 문서 기준).

- `system.metrics` — 순간값
- `system.events` — 쿼리 처리 누적 통계(서버 시작 이후 누적 카운터)
- `system.asynchronous_metrics` — 주기적으로 계산되는 하드웨어·집계 상태

이 셋을 한꺼번에 외부로 흘려보내는 통로가 **Prometheus 엔드포인트**다.
ClickHouse는 `config.xml`의 `<prometheus>` 섹션을 켜면 `/metrics` HTTP 엔드포인트를 노출하고, 기본 포트는 9363이다(소스 설정 예시 기준).
이 엔드포인트가 위 세 테이블의 값을 Prometheus 포맷으로 내보내므로, Grafana 같은 데서 시계열 대시보드와 알람을 붙일 수 있다.

ClickHouse Cloud를 쓴다면 더 간단하다.
`$HOST:$PORT/dashboard` 경로에 QPS, CPU, 메모리, 머지 등을 보여주는 내장 관측 대시보드가 기본 제공된다.

> 참고로 `system.events`의 구체 컬럼·대표 이벤트 목록과 Keeper 자체 모니터링 지표(4lw `mntr` 등)는 이번 정리 범위 밖이라, 깊게 쓰려면 각 공식 문서를 별도로 확인하는 게 좋다.

## "왜 이렇게?" — 운영을 SQL로 만든다는 선택

마지막으로 한 걸음 물러서 보자.
왜 ClickHouse는 서버 상태를 굳이 *테이블*로 노출할까?

답은 일관성이다.
ClickHouse가 잘하는 단 하나는 "대량 데이터를 SQL로 빠르게 집계"하는 것이다.
그렇다면 운영 데이터(파트 수, 쿼리 로그, 에러 카운트)도 결국 같은 모양의 데이터다.
별도의 관리 콘솔이나 전용 프로토콜을 만드는 대신, *이미 가진 무기*로 운영까지 처리하게 한 것이다.
운영자는 `system.parts`를 `GROUP BY partition` 하고, `system.query_log`를 `ORDER BY query_duration_ms` 한다 — 새로 배울 게 없다.

그래서 ClickHouse 운영의 절반은 "어떤 system 테이블을, 어떤 컬럼으로 보느냐"를 아는 것이다.
이 장의 표와 쿼리가 그 지도 역할을 한다.

다음 14장에서는 이런 운영 원칙들이 실제 대기업 환경에서 어떻게 적용됐는지 — 공개된 레퍼런스 사례를 본다.

## 참고

- ClickHouse Docs — system.merges: https://clickhouse.com/docs/operations/system-tables/merges
- ClickHouse Docs — system.mutations: https://clickhouse.com/docs/operations/system-tables/mutations
- ClickHouse Docs — system.query_log: https://clickhouse.com/docs/operations/system-tables/query_log
- ClickHouse Docs — system.metrics: https://clickhouse.com/docs/operations/system-tables/metrics
- ClickHouse Docs — system.asynchronous_metrics: https://clickhouse.com/docs/operations/system-tables/asynchronous_metrics
- ClickHouse Docs — system.replicas: https://clickhouse.com/docs/operations/system-tables/replicas
- ClickHouse Docs — system.errors: https://clickhouse.com/docs/operations/system-tables/errors
- ClickHouse Docs — Monitoring: https://clickhouse.com/docs/operations/monitoring
- ClickHouse KB — Exception: Too many parts: https://clickhouse.com/docs/knowledgebase/exception-too-many-parts
- ClickHouse Blog — Common getting started issues: https://clickhouse.com/blog/common-getting-started-issues-with-clickhouse
- ClickHouse/ClickHouse 소스 (MergeTreeSettings, config 예시): https://github.com/ClickHouse/ClickHouse
- Altinity KB — Memory configuration settings: https://kb.altinity.com/altinity-kb-setup-and-maintenance/altinity-kb-memory-configuration-settings/
- Trigger.dev — ClickHouse too-many-parts postmortem: https://trigger.dev/blog/clickhouse-too-many-parts-postmortem
