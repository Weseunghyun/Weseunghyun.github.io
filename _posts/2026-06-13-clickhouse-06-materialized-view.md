---
title: "ClickHouse 완전 정복 6장 — Materialized View: 쿼리 시점이 아니라 삽입 시점에 미리 계산한다"
date: 2026-06-13 23:53:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "심화"]
---
3장에서 MergeTree의 part는 한 번 만들어지면 고치지 않는 불변 객체라고 했다.
4장에서는 sparse primary index가 granule 단위로 데이터를 건너뛰며 읽는 구조를 봤다.
이 둘 덕분에 ClickHouse는 큰 테이블을 통째로 빠르게 훑는다.
하지만 "매번" 전체를 훑는 건, 자주 던지는 집계 쿼리에는 여전히 낭비다.
하루에 수백 번 "오늘 사용자별 평균 점수"를 묻는데, 그때마다 원본 수억 행을 다시 집계할 이유가 없다.

ClickHouse의 Materialized View(MV)는 바로 이 낭비를 없애는 장치다.
한 줄로 핵심을 던지면 이렇다.

> ClickHouse의 (증분) Materialized View는 "저장된 쿼리 결과"가 아니라, 데이터가 들어올 때 작동하는 **삽입 트리거(insert trigger)**다.

공식 문서도 같은 표현을 쓴다.
"a ClickHouse materialized view is just a trigger that runs a query on blocks of data as they're inserted into a table." (공식 문서 기준)
즉 계산 비용을 **쿼리 시점에서 삽입 시점으로 옮기는** 것이 MV의 본질이다(shift the cost of computation from query time to insert time).

## 비유: 영수증을 정리하는 직원

이건 이해를 돕는 비유다 — 실제 동작은 바로 아래 메커니즘에서 다시 정확히 짚는다.

가게 계산대에 영수증(원본 INSERT)이 한 장씩 쌓인다고 하자.
일반적인 RDB의 뷰(view)는 "결산이 필요할 때마다" 그동안 쌓인 영수증을 전부 다시 세는 직원이다.
영수증이 수억 장이면 매번 다시 세는 게 느리다.

ClickHouse의 증분 MV는 다르다.
새 영수증이 들어올 때마다 그 한 장을 보고 "오늘 매출 장부"에 즉시 더해 두는 직원이다.
결산할 때는 이미 더해진 장부만 보면 되니 빠르다.

대신 이 직원에겐 결정적인 특징이 하나 있다.
**이미 쌓여 있던 과거 영수증은 보지 않는다.**
이 직원이 출근한(=MV를 만든) 시점부터 들어오는 영수증만 장부에 반영한다.
이 "과거를 안 본다"는 성질이 뒤에서 다룰 함정의 핵심이다.

## 메커니즘 1 — 삽입 트리거와 타깃 테이블

증분 MV의 동작은 단순하다.

소스 테이블에 새 데이터 블록이 INSERT되면 MV의 SELECT가 그 블록에만 실행되고,
결과가 별도의 **타깃 테이블**로 다시 INSERT된다.
이때 새로 들어온 블록만 처리하고, 기존/과거 데이터는 건드리지 않는다(공식 문서 기준).

![](/assets/img/posts/clickhouse-06-materialized-view/clickhouse-materialized-view.svg)

타깃 테이블은 `TO` 절로 명시한다.

```sql
CREATE MATERIALIZED VIEW view_name TO target_table AS
SELECT ... FROM source_table;
```

`TO` 절로 타깃을 지정하면 MV는 그 타깃 테이블에 결과를 적재한다.
타깃 테이블은 보통 SummingMergeTree나 AggregatingMergeTree 같은 집계용 엔진을 쓴다 — 삽입 시점마다 들어오는 부분 집계를 계속 누적하기 위해서다.
(참고로 `TO` 절을 쓰면 뒤에서 설명할 POPULATE는 함께 쓸 수 없다.)

## 메커니즘 2 — 과거 데이터는 어떻게 채우나 (POPULATE의 함정)

MV는 "만든 이후의 INSERT"만 본다.
그러면 이미 테이블에 쌓여 있던 과거 데이터는 어떻게 집계에 넣을까?

가장 먼저 떠오르는 게 `POPULATE` 키워드다.
MV 생성 시 소스 테이블의 기존 데이터를 한 번 훑어 MV에 채워 넣는다.
하지만 공식 문서는 POPULATE 사용을 권장하지 않는다(공식 문서 기준).

이유는 이렇다.
POPULATE는 `CREATE TABLE ... AS SELECT`처럼 동작하는데,
POPULATE 작업이 진행되는 동안 소스 테이블에 들어온 행은 MV에서 **누락될 수 있다**.
또 POPULATE는 Replicated 데이터베이스나 ClickHouse Cloud에서 지원되지 않고, `TO [db].[table]` 절과 함께 쓸 수도 없다.

권장 방식은 명시적인 백필이다.
즉 `INSERT INTO ... SELECT` 문으로 소스(또는 s3() 같은 외부 소스)에서 데이터를 다시 흘려보내, MV 트리거를 정상적으로 발동시키는 것이다.

```sql
-- POPULATE 대신: 과거 구간을 직접 다시 흘려보내 트리거 발동
INSERT INTO hourly_counts
SELECT toStartOfHour(event_time), count(), sum(payload_bytes)
FROM events
WHERE event_time < now()
GROUP BY 1;
```

이 방식은 POPULATE의 누락 위험을 피하면서 같은 결과를 만든다.

| 백필 방식 | 동작 | 누락 위험 | Cloud/Replicated |
|---|---|---|---|
| `POPULATE` | 생성 시 기존 데이터 1회 채움 | 작업 중 들어온 행 누락 가능 | 미지원 |
| `INSERT ... SELECT` | 과거 구간을 다시 흘려보내 트리거 발동 | 없음(권장) | 지원 |

## 메커니즘 3 — 집계 상태를 누적한다 (-State / -Merge)

여기서 한 가지 의문이 생긴다.
삽입 블록마다 부분 집계를 만들어 더하는 건, 합계(sum)나 카운트(count)처럼 **그냥 더하면 되는** 값이면 쉽다.
이런 가산(summable) 집계는 SummingMergeTree 타깃이면 충분하다 — 같은 키의 행을 머지하면서 자동으로 더해 준다.

문제는 평균(avg), 고유 개수(uniq), 분위수(quantile)처럼 **단순히 더할 수 없는** 집계다.
일별 평균 두 개를 그냥 더한다고 전체 평균이 되지 않는다.
이걸 위해 ClickHouse는 집계 함수에 붙이는 접미사 두 개를 쓴다.

- `-State`(예: `avgState`, `uniqState`, `quantileState`): 최종 값이 아니라 **중간 집계 상태**를 만든다.
  공식 문서 표현으로는 "an intermediate state of the aggregation"이며 타입은 `AggregateFunction(...)`이다(공식 문서 기준).
  avg라면 "합과 개수"처럼, 나중에 합칠 수 있는 형태로 들고 있는 셈이다.
- `-Merge`(예: `avgMerge`, `uniqMerge`, `quantileMerge`): 저장된 중간 상태들을 합쳐 **최종 값**을 돌려준다.

따라서 패턴은 항상 같다.
**MV에서는 `-State`로 저장하고, 조회할 때 `-Merge`로 읽는다.**
저장 엔진은 이런 상태를 누적·머지하는 AggregatingMergeTree를 쓴다.

```sql
-- 1) 집계 상태를 저장할 타깃 테이블 (AggregatingMergeTree)
CREATE TABLE daily_stats
(
    day        Date,
    avg_score  AggregateFunction(avg, Int32),
    p999_score AggregateFunction(quantile(0.999), Int32),
    uniq_users AggregateFunction(uniq, UInt64)
)
ENGINE = AggregatingMergeTree
ORDER BY day;

-- 2) MV: 삽입되는 블록마다 -State 로 중간 집계 상태를 만들어 타깃에 적재
CREATE MATERIALIZED VIEW daily_stats_mv TO daily_stats AS
SELECT
    toDate(event_time)          AS day,
    avgState(score)             AS avg_score,
    quantileState(0.999)(score) AS p999_score,
    uniqState(user_id)          AS uniq_users
FROM events
GROUP BY day;

-- 3) 조회 시 -Merge 로 중간 상태들을 최종값으로 합산
SELECT
    day,
    avgMerge(avg_score)              AS avg_score,
    quantileMerge(0.999)(p999_score) AS p999_score,
    uniqMerge(uniq_users)            AS uniq_users
FROM daily_stats
GROUP BY day
ORDER BY day;
```

합/카운트만 필요하면 굳이 `-State`까지 갈 필요 없이 SummingMergeTree로 충분하다.

```sql
CREATE TABLE hourly_counts
(
    hour  DateTime,
    cnt   UInt64,
    bytes UInt64
)
ENGINE = SummingMergeTree
ORDER BY hour;

CREATE MATERIALIZED VIEW hourly_counts_mv TO hourly_counts AS
SELECT
    toStartOfHour(event_time) AS hour,
    count()                   AS cnt,
    sum(payload_bytes)        AS bytes
FROM events
GROUP BY hour;
```

정리하면, 가산 집계는 SummingMergeTree, 비가산 집계는 AggregatingMergeTree + `-State`/`-Merge`다.

## 두 가지 함정 — 연쇄와 JOIN

증분 MV를 쓸 때 꼭 알아야 할 성질이 둘 있다.

첫째, **연쇄(cascading)가 된다.**
한 MV의 타깃 테이블이 다음 MV의 소스 테이블이 될 수 있다.
이렇게 MV를 체인으로 엮으면 "원본 → 시간별 집계 → 일별 집계"처럼 여러 단계 변환을 단계적으로 흘려보낼 수 있다(공식 문서 기준).

둘째, **JOIN의 함정이다.**
MV는 쿼리에서 **가장 왼쪽(소스) 테이블에 대한 INSERT에만** 트리거된다.
JOIN의 우측 테이블 데이터가 바뀌어도 MV는 트리거되지 않는다.
공식 문서도 "Right-side tables in JOINs don't trigger updates, even if their data changes."라고 못박는다(공식 문서 기준).

이게 왜 함정이냐면,
예를 들어 주문(orders)을 소스로 두고 상품(products) 디멘션 테이블을 JOIN해 이름을 붙이는 MV가 있다고 하자.
새 주문이 들어오면 그 시점의 products 스냅샷으로 이름이 붙는다.
그런데 나중에 상품 이름이 바뀌어도, products만 바뀐 것으로는 MV가 다시 돌지 않는다.
이미 적재된 행은 옛 이름 그대로 남는다.
디멘션이 자주 바뀐다면 증분 MV + JOIN은 위험하다.

이 한계를 풀어 주는 게 다음에 볼 Refreshable MV다.

## 다른 종류 — Refreshable Materialized View

지금까지의 증분 MV는 "삽입될 때마다 조금씩" 갱신하는 방식이다.
반면 **Refreshable MV**는 정해진 주기마다 **전체 데이터셋에 대해 쿼리를 통째로 다시 실행**하고, 그 결과를 타깃 테이블에 저장한다.
공식 문서 표현으로 "the periodic execution of the query over the full dataset"다(공식 문서 기준).

이건 위에서 본 JOIN 함정처럼 증분이 부적합한 경우(복잡한 JOIN으로 만든 일·주 단위 리포트 등)에 적합하다.
다만 문서는 여전히 증분 MV를 기본으로 권장한다 — "incremental materialized views are enormously powerful and typically scale much better"라고 적는다(공식 문서 기준).
즉 Refreshable는 "증분이 안 맞을 때의 대안"이지, 더 우월한 상위 호환이 아니다.

핵심 동작은 atomic 교체다.
refresh가 끝나면 결과 집합이 "atomically and transparently" 갱신되어(공식 문서 기준), 조회 측은 항상 일관된 스냅샷을 본다 — 중간 상태가 비치지 않는다.

문법은 두 갈래다.

- `REFRESH EVERY`: 시각에 정렬해 실행(예: 매일 자정).
- `REFRESH AFTER`: 이전 refresh가 끝난 뒤 상대 시간만큼 지나 실행(시각 정렬 없음).

```sql
CREATE TABLE top_products (product_id UInt64, name String, revenue Decimal(18,2))
ENGINE = MergeTree ORDER BY product_id;

-- 매일 02:00(±30분 분산)에 전체 쿼리 재실행 → top_products 전체를 atomic 교체
CREATE MATERIALIZED VIEW top_products_mv
REFRESH EVERY 1 DAY OFFSET 2 HOUR RANDOMIZE FOR 1 HOUR
TO top_products AS
SELECT o.product_id, p.name, sum(o.amount) AS revenue
FROM orders AS o
INNER JOIN products AS p ON p.id = o.product_id   -- 증분 MV였다면 p 변경은 트리거 안 됨
GROUP BY o.product_id, p.name
ORDER BY revenue DESC
LIMIT 100;
```

여기 쓰인 옵션 몇 개를 짚어 두자.

| 옵션 | 의미 |
|---|---|
| `REFRESH EVERY` | 지정 주기마다 시각 정렬해 실행 |
| `REFRESH AFTER` | 이전 refresh 완료 후 상대 시간 뒤 실행 |
| `APPEND` | 교체 대신 새 행을 끝에 추가(스냅샷 누적) |
| `RANDOMIZE FOR` | refresh 시각을 무작위로 흔들어 부하 spike 분산 |
| `DEPENDS ON` | 다른 MV refresh 완료 후에만 실행(의존성) |

`APPEND`는 기본 동작(전체 교체)과 정반대다.
refresh 때 기존 행을 지우지 않고 새 행을 끝에 추가한다(공식 문서 기준).
"매 10초의 큐 깊이"처럼 시간에 따른 스냅샷을 쌓는 패턴에 쓴다.

```sql
CREATE TABLE queue_depth_snapshots (ts DateTime, depth UInt64)
ENGINE = MergeTree ORDER BY ts;

CREATE MATERIALIZED VIEW queue_depth_mv
REFRESH EVERY 10 SECOND APPEND      -- APPEND: 교체 대신 끝에 추가
TO queue_depth_snapshots AS
SELECT now() AS ts, count() AS depth FROM pending_jobs;

-- 주기 변경도 가능
ALTER TABLE queue_depth_mv MODIFY REFRESH AFTER 30 MINUTE;
```

`DEPENDS ON`은 한 발 더 나간다.
refreshable MV끼리 의존성을 걸면, 한 MV가 다른 MV의 refresh가 끝난 뒤에만 실행된다(공식 문서 기준).
"원본 집계가 끝난 다음 그 위에 리포트를 만든다" 같은 단순한 DAG/스케줄 워크플로를 ClickHouse 안에서 표현할 수 있다.

운영에서는 refresh가 잘 돌고 있는지 봐야 한다.
상태·마지막/다음 refresh 시각·읽고 쓴 행 수는 `system.view_refreshes`로 확인한다(13장 운영·모니터링에서 system 테이블을 더 다룬다).

```sql
SELECT view, status, last_success_time, next_refresh_time, read_rows, written_rows
FROM system.view_refreshes;
```

## 성숙도 한 가지 — Refreshable MV는 언제부터 안정인가

Refreshable MV는 처음 도입될 때 experimental 기능이었다.
공개적으로 확인되는 범위에선 23.12에서 experimental로 도입되었고,
이후 PR #70550 "Refreshable materialized views are not experimental anymore"(2024-10-11 머지)로 experimental 플래그가 제거되어 일반 사용 가능해졌다.
머지 타이밍으로 보면 24.10 릴리스 부근으로 보이지만, 해당 changelog 한 줄을 직접 확인하지는 못해 정확한 릴리스 버전은 단정하지 않는다.
실무에서 쓸 때는 "내가 쓰는 버전에서 실험 설정 없이 동작하는지"를 직접 확인하고, 가능하면 최신 stable/LTS(2026-06 기준 stable 26.4 · LTS 26.3)에서 쓰는 편이 안전하다.

## "왜 이렇게?" — 삽입 시점으로 비용을 미는 이유

마지막으로 한 번 더 질문해 보자.
왜 ClickHouse는 결과를 따로 저장(materialize)하지 않고, 굳이 "삽입 트리거" 모델을 택했을까?

3장에서 본 것처럼 ClickHouse의 part는 불변이고, 데이터는 보통 대량 배치로 추가된다(append).
이 append-heavy한 세계에서는, 들어오는 그 순간 블록 단위로 부분 집계를 한 번 계산해 두는 게 자연스럽다.
이미 한 번 손에 들고 있는 데이터를 그때 처리하면, 같은 데이터를 쿼리 때마다 다시 디스크에서 읽어 집계하는 낭비를 없앨 수 있다.
이것이 "비용을 쿼리 시점에서 삽입 시점으로 옮긴다"는 말의 실체다.

반대로 그 대가가 두 함정(과거 데이터 미반영, JOIN 우측 미트리거)이다.
이 대가가 받아들이기 어려운 경우 — 전체 재계산이 필요한 복잡한 리포트 — 를 위해 Refreshable MV가 별도로 존재한다.
즉 ClickHouse는 "삽입 시 증분"을 기본으로 두고, "주기적 전체 재계산"을 예외로 두는 식으로 두 모델을 나눠 제공한다.

다음 7장에서는 또 다른 사전 계산 장치인 **Projection**을 본다.
MV가 별도 타깃 테이블에 결과를 쌓는 반면, Projection은 같은 테이블 안에 정렬·집계가 다른 또 하나의 데이터 사본을 두는 방식이다.
둘은 자주 비교되니, 이번 장의 "삽입 시점 계산"이라는 개념을 들고 넘어가면 차이가 선명해진다.

## 참고

- ClickHouse Docs — Incremental Materialized View: https://clickhouse.com/docs/materialized-view/incremental-materialized-view
- ClickHouse Docs — Materialized View 개요: https://clickhouse.com/docs/materialized-view
- ClickHouse Docs — Refreshable Materialized View: https://clickhouse.com/docs/materialized-view/refreshable-materialized-view
- ClickHouse Docs — CREATE VIEW (구문): https://clickhouse.com/docs/sql-reference/statements/create/view
- ClickHouse Docs — Aggregate function combinators (-State/-Merge): https://clickhouse.com/docs/en/sql-reference/aggregate-functions/combinators
- ClickHouse GitHub PR #70550 — Refreshable materialized views are not experimental anymore: https://github.com/ClickHouse/ClickHouse/pull/70550
- ClickHouse Blog — Release 23.12 (Refreshable MV 도입): https://clickhouse.com/blog/clickhouse-release-23-12
