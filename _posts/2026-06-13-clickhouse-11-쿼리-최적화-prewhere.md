---
title: "ClickHouse 완전 정복 11장 — 쿼리 최적화: PREWHERE와 EXPLAIN으로 읽는 양 줄이기"
date: 2026-06-13 23:48:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "운영"]
---
지금까지 우리는 데이터가 디스크에 어떻게 누워 있는지를 봤다.
2장에서 컬럼 단위 저장과 압축·벡터화를,
3장에서 MergeTree의 불변 part와 머지를,
4장에서 sparse primary index가 granule을 어떻게 솎아내는지를 다뤘다.

이번 장의 주제는 "그래서 쿼리 한 방을 어떻게 빠르게 만드느냐"다.
ClickHouse 쿼리 최적화의 핵심은 거의 항상 한 문장으로 요약된다.
**디스크에서 읽는 양을 줄여라.**
컬럼 DB에서 느린 쿼리의 대부분은 "필요 없는 데이터를 너무 많이 읽는" 데서 온다.

이 장에서는 그 "읽는 양"을 줄이는 도구 세 가지를 본다.
PREWHERE(필터를 먼저 읽기), optimize_read_in_order(정렬을 건너뛰기), 그리고 EXPLAIN(무엇이 실제로 일어나는지 들여다보기)다.
마지막으로 system.query_log로 느린 쿼리를 찾아내는 법까지 본다.

---

## 가장 큰 레버리지는 여전히 ORDER BY다

먼저 김을 빼자.
PREWHERE보다, EXPLAIN보다, 그 무엇보다 큰 레버리지는 3·4장에서 본 **primary index(=ORDER BY) 설계**다.

ClickHouse 공식 엔지니어링 가이드는 적절한 ORDER BY 선택만으로 쿼리 성능을 100배 이상 끌어올릴 수 있다고 말한다(공식 문서 기준).
가이드의 예시 하나를 보자.
`town = 'LONDON'` 조건으로 거를 때, 정렬키가 맞지 않으면 약 2,764만 행을 훑어 44ms가 걸리지만,
ORDER BY를 town에 맞춰 설계하면 같은 결과를 81,920행만 읽고 5ms에 끝낸다(공식 문서 기준).
약 8.8배 차이다.

반대로 말하면, **primary key에 없는 컬럼만으로 필터링하면 풀 스캔이 일어난다.**
그래서 이번 장의 모든 기법은 "좋은 ORDER BY가 이미 깔려 있다"는 전제 위에서 그 위에 얹는 미세 조정에 가깝다.
순서가 틀린 정렬키를 PREWHERE로 구제할 수는 없다.

---

## PREWHERE — 필터에 쓰는 컬럼만 먼저 읽는다

### 비유부터

비유 하나를 던지자.
도서관에서 "2026년에 출간된, 제목에 'ClickHouse'가 들어간 책"을 찾는다고 하자.
방법 A: 모든 책을 한 권씩 다 꺼내서, 출간연도도 보고 제목도 본다.
방법 B: 먼저 얇은 "출간연도 목록"만 훑어서 2026년 칸을 추리고, 그 칸의 책들만 꺼내서 제목을 확인한다.

PREWHERE는 방법 B다.
이건 이해를 돕는 비유이고, 실제로는 "책"이 아니라 granule(4장의 8192행짜리 묶음)과 컬럼 파일 단위로 동작한다.

### 메커니즘

ClickHouse의 WHERE는 보통 이렇게 동작한다.
필요한 SELECT 컬럼과 필터 컬럼을 다 읽어 들인 뒤, 그 위에서 조건을 평가한다.

PREWHERE는 단계를 쪼갠다(공식 문서 기준).
1. 먼저 **필터 표현식에 등장하는 컬럼만** 디스크에서 읽는다.
2. 그 컬럼들로 조건을 평가해서, 조건이 참인 행이 하나라도 들어 있는 granule만 골라낸다.
3. 그렇게 살아남은 granule에 대해서만 나머지 SELECT 컬럼을 읽는다.

공식 문서의 표현을 빌리면, 각 단계마다 "직전 필터를 통과한 행이 최소 하나라도 있는 granule만 다음 컬럼에서 로드한다".
필터에 쓰는 컬럼이 전체 쿼리 컬럼의 일부분이고, 많은 granule이 탈락한다면, 디스크에서 읽는 데이터가 크게 줄어든다.

![](/assets/img/posts/clickhouse-11-쿼리-최적화-prewhere/clickhouse-prewhere.svg)

공식 최적화 문서는 동일한 쿼리에서 처리하는 행수(2.31M)는 같은데도,
디스크에서 읽은 데이터가 23.36MB에서 6.74MB로 약 3배 이상 줄어든 예를 보여준다(공식 문서 기준).
행수가 같다는 게 포인트다 — PREWHERE는 "무엇을 계산하느냐"를 바꾸지 않는다.
"무엇을 디스크에서 읽느냐"를 바꾼다.

### WHERE와 PREWHERE의 관계

문법은 똑같다.
공식 문서는 "차이는 테이블에서 어떤 데이터를 읽는가에 있다"고 못박는다.
한 쿼리에 PREWHERE와 WHERE가 동시에 존재할 수 있고, 이때 PREWHERE가 WHERE보다 먼저 실행된다.

그리고 중요한 제약: **PREWHERE는 *MergeTree 계열 테이블에서만 지원된다**(공식 문서 기준).
3장에서 본 그 MergeTree 패밀리 말이다.

### 보통은 자동으로 일어난다

좋은 소식은, 이 최적화가 기본으로 켜져 있다는 것이다.
`optimize_move_to_prewhere` 설정이 그것이고, 기본값은 1(true)이다.
공식 문서는 이 자동 활성을 23.2 버전 기준으로 명시한다(공식 문서 기준).
이 값을 0으로 두면 WHERE의 일부 조건을 PREWHERE로 자동 이동시키는 휴리스틱이 꺼진다.

자동 이동을 담당하는 건 내부의 MergeTreeWhereOptimizer다.
이 옵티마이저는 조건의 추정 선택도(얼마나 강하게 거르는가)와 관련 컬럼의 크기를 보고, 어떤 조건을 PREWHERE로 옮길지·어떤 순서로 평가할지를 정한다.
원칙은 "작고 강하게 거르는 조건을 먼저"다.
다만 정렬 기준이 압축 크기인지 비압축 크기인지는 출처마다 표현이 조금 엇갈리므로, 여기서는 "작은·선택도 높은 조건을 먼저 본다"는 원칙만 확정 사실로 두자.

### 그럼 수동 PREWHERE는 언제?

자동이 잘 되는데 왜 직접 쓰는가.
옵티마이저가 옮기지 않는 조건들이 있기 때문이다.
비결정적 함수(예: `rand()`), 상수식, ARRAY JOIN 대상 컬럼, GLOBAL IN 등을 포함한 조건은 자동 이동 대상에서 빠진다(공식 문서·소스 기준).
또 자동 선택이 항상 최적은 아니다 — 작지만 아주 강하게 거르는 컬럼이 있다면, 그걸 직접 PREWHERE에 명시하는 게 나을 때가 있다.

```sql
-- 자동 이동(optimize_move_to_prewhere=1)이 최적이 아닐 때 명시적으로 지정
SELECT user_id, event_name, payload   -- payload는 큰 컬럼
FROM events
PREWHERE status = 5                   -- 강하게 필터링하는 작은 컬럼 먼저
WHERE   event_date >= '2026-06-01';
-- PREWHERE는 *MergeTree 계열에서만 지원
```

여기서 의도는 명확하다.
`status`는 작은 정수 컬럼이고 `status = 5`가 행을 강하게 걸러낸다고 치자.
그러면 먼저 `status`만 읽어 granule을 솎아낸 뒤, 큰 `payload` 컬럼은 살아남은 granule에 대해서만 읽는다.

한 가지 함정: FINAL 쿼리(머지 안 끝난 데이터를 쿼리 시점에 합치는 변종 테이블 조회)에서는,
`optimize_move_to_prewhere`와 **별개로** `optimize_move_to_prewhere_if_final`도 켜야 자동 이동이 올바른 결과로 적용된다(공식 문서·소스 기준).

---

## optimize_read_in_order — 정렬을 아예 건너뛰기

두 번째 도구.
정렬(ORDER BY)은 비싸다.
보통은 데이터를 다 읽어 메모리에 올린 뒤 정렬해야 한다.

그런데 쿼리의 ORDER BY 컬럼이 테이블 primary key의 **접두사(prefix)**라면 어떨까.
데이터는 이미 디스크에 그 순서대로 누워 있다(3·4장).
그러면 따로 정렬할 필요 없이, 디스크 순서대로 읽기만 하면 결과가 정렬되어 나온다.
이게 `optimize_read_in_order`다(공식 문서 기준).

PRIMARY KEY와 ORDER BY를 다르게 지정한 테이블이라도, primary key가 ORDER BY의 접두사이기만 하면 적용된다.

이 최적화가 특히 빛나는 건 LIMIT가 붙을 때다.
정렬 순서로 읽으니, 필요한 만큼만 읽고 조기에 멈출 수 있다 — 전체를 메모리에 올려 정렬하지 않으므로 메모리 사용도 적다(공식 문서 기준).
다만 공개적으로 알려진 트레이드오프가 하나 있다.
정렬 순서를 유지하며 읽어야 하므로, 테이블을 읽는 병렬성이 줄어드는 경향이 있다는 점이다.
즉 "정렬 비용을 아끼는 대신 읽기 병렬성을 일부 양보"하는 교환이다.

---

## EXPLAIN — 추측 대신 들여다보기

여기까지는 "이렇게 하면 빨라진다"는 이야기였다.
하지만 최적화의 철칙은 **측정 먼저**다.
정말 인덱스가 걸렸는지, 정말 PREWHERE로 옮겨졌는지, 정말 병렬로 도는지 — 추측하지 말고 EXPLAIN으로 본다.

ClickHouse EXPLAIN에는 여러 변형이 있다(공식 문서 기준).

| 변형 | 무엇을 보여주나 |
|---|---|
| `EXPLAIN AST` | 파싱된 추상 구문 트리 |
| `EXPLAIN SYNTAX` | 재작성(rewrite)된 쿼리 |
| `EXPLAIN QUERY TREE` | 쿼리 트리 표현 |
| `EXPLAIN PLAN` | 논리적 실행 단계 |
| `EXPLAIN PIPELINE` | 프로세서·스레드 기반 물리 실행 그래프 |
| `EXPLAIN ESTIMATE` | MergeTree의 part·mark·row 추정치 |

실무에서 가장 자주 쓰는 두 가지를 보자.

### EXPLAIN PLAN indexes=1 — 인덱스가 정말 걸렸나

`EXPLAIN`에 `indexes = 1` 옵션을 주면, 사용된 인덱스·필터된 part 수·필터된 granule 수를 보여준다(공식 문서 기준, MergeTree 전용·기본값 0).
이게 핵심이다 — primary index나 skip index가 **실제로 데이터를 걸러내고 있는지**를 눈으로 확인할 수 있다.

```sql
EXPLAIN indexes = 1
SELECT count()
FROM hits
WHERE town = 'LONDON';
-- 출력에 사용된 인덱스, 필터된 parts 수, 필터된 granules 수가 표시됨
```

만약 필터된 granule 수가 거의 0이라면(즉 거의 다 읽는다면), 그 인덱스는 이 쿼리에 도움이 안 되고 있다는 신호다.
관련 옵션으로 `actions=1`(단계 상세), `projections=1`, `json=1` 등도 있다.

### EXPLAIN PIPELINE — 병렬로 도는가

`EXPLAIN PIPELINE`은 실행 파이프라인을 프로세서 단위로 보여준다(공식 문서 기준).

```sql
EXPLAIN PIPELINE
SELECT town, count()
FROM hits
GROUP BY town;
```

출력에는 `MergeTreeThread`, `FilterTransform`, `AggregatingTransform` 같은 프로세서 이름들이 나오고,
보조 자료에 따르면 각 프로세서 뒤의 `× N` 표기는 N개의 병렬 스레드를 의미하며,
마지막에 `Resize N -> 1` 단계에서 병렬 스트림들이 하나로 합쳐진다.
대략적인 해석은 이렇다 — `× 8`처럼 큰 숫자가 보이면 잘 병렬화된 것이고,
어디서나 `× 1`만 보인다면 병렬성이 거의 안 쓰이고 있다는 신호로 읽을 수 있다.
(이 `× N` 표기와 스레드 수 해석의 세부는 공개적으로 확인되는 범위에선 보조 아티클 쪽 설명이라, 자기 환경의 버전에서 한 번 직접 확인해 보길 권한다.)

---

## 느린 쿼리는 어디서 찾나 — system.query_log

마지막 도구.
"어떤 쿼리가 느린지"를 모르면 최적화할 대상도 없다.
ClickHouse는 모든 쿼리 실행을 `system.query_log` 테이블에 기록한다(공식 문서 기준).

핵심 컬럼들은 이렇다.

| 컬럼 | 의미 |
|---|---|
| `query_duration_ms` | 실행 시간 |
| `read_rows` | 읽은 행 수 |
| `read_bytes` | 읽은 바이트 |
| `memory_usage` | 메모리 사용량 |
| `normalized_query_hash` | 반복되는 같은 형태의 쿼리 식별 |

느린 쿼리 상위 N개는 이렇게 뽑는다.

```sql
SELECT
    query_duration_ms,
    read_rows,
    formatReadableSize(read_bytes) AS read_bytes,
    formatReadableSize(memory_usage) AS mem,
    normalized_query_hash,
    query
FROM system.query_log
WHERE type = 'QueryFinish'
  AND event_time >= now() - INTERVAL 1 DAY
  AND is_initial_query = 1
ORDER BY query_duration_ms DESC
LIMIT 20;
```

`type = 'QueryFinish'`로 완료된 쿼리만, `is_initial_query = 1`로 사용자가 직접 보낸 쿼리만 거른 뒤, 실행 시간 내림차순으로 정렬한다.

여기서 봐야 할 핵심 신호는 **결과 대비 read_rows**다.
1억 행을 읽고 10행을 반환했다면, 그건 인덱스가 안 걸렸거나 스키마/쿼리 설계가 잘못됐다는 강한 신호다.
앞서 본 `EXPLAIN indexes=1`로 그 쿼리를 다시 분석할 차례다.
로그 노이즈를 줄이고 싶으면 `log_queries_min_query_duration_ms`로 기록 임계값을 둘 수 있다(공식 문서 기준).
`normalized_query_hash`로 묶으면, 같은 모양의 쿼리가 반복적으로 느린지도 한눈에 본다.

---

## 흔한 안티패턴 둘

기법을 다 알아도, 기본기를 어기면 무너진다.

**SELECT \*** — 컬럼 DB에서 `SELECT *`는 모든 컬럼 파일을 디스크에서 읽게 만든다(공식 문서 기준).
2장에서 본 컬럼 저장의 장점("필요한 컬럼만 읽는다")을 스스로 버리는 셈이다.
필요한 컬럼만 명시하라.

**Nullable 남용** — Nullable 컬럼은 값이 NULL인지 추적하는 별도의 UInt8 컬럼이 따라붙어 오버헤드가 생긴다(공식 문서 기준).
정말 NULL과 "0/빈 문자열"을 구분해야 하는 게 아니라면, 합리적인 기본값(빈 문자열, 0, -1 등)을 쓰는 편이 낫다.
5장의 스키마·데이터타입 이야기와 이어지는 대목이다.

---

## 정리하며 — "왜 이렇게 설계됐을까"

마지막으로 질문 하나.
왜 ClickHouse는 WHERE를 그냥 잘 처리하지 않고 PREWHERE라는 별도 개념까지 두었을까.

답은 이 DB의 본질에 있다.
컬럼 DB의 비용은 거의 전부 "디스크에서 컬럼을 읽는 비용"이다.
같은 결과를 내더라도, 읽는 컬럼·읽는 granule을 줄이면 그대로 속도가 된다.
PREWHERE는 "필터를 먼저 평가해서 읽을 범위를 좁힌다"는 단 하나의 아이디어를, 컬럼 저장 구조 위에서 극단까지 밀어붙인 결과다.

그래서 이 장의 도구들은 결국 한 방향을 가리킨다.
좋은 ORDER BY로 읽을 part·granule을 줄이고(3·4장),
PREWHERE로 읽을 컬럼을 줄이고,
read_in_order로 정렬 단계를 줄이고,
EXPLAIN과 query_log로 그게 정말 됐는지 확인한다.

다음 12장에서는 데이터를 "언제 지울 것인가" — TTL과 저장 관리로 넘어간다.
오래된 데이터를 자동으로 만료시키고, 저장 계층을 옮기는 방법을 본다.

---

## 참고

- ClickHouse 공식 문서 — PREWHERE: <https://clickhouse.com/docs/sql-reference/statements/select/prewhere>
- ClickHouse 공식 문서 — Optimize / PREWHERE 동작: <https://clickhouse.com/docs/optimize/prewhere>
- ClickHouse 공식 문서 — ORDER BY / optimize_read_in_order: <https://clickhouse.com/docs/sql-reference/statements/select/order-by>
- ClickHouse 공식 문서 — EXPLAIN: <https://clickhouse.com/docs/sql-reference/statements/explain>
- ClickHouse 공식 문서 — Data skipping indexes: <https://clickhouse.com/docs/optimize/skipping-indexes>
- ClickHouse 공식 문서 — system.query_log: <https://clickhouse.com/docs/operations/system-tables/query_log>
- ClickHouse 엔지니어링 가이드 — Query optimization (definitive guide): <https://clickhouse.com/resources/engineering/clickhouse-query-optimisation-definitive-guide>
- ClickHouse/ClickHouse 소스 (DeepWiki): <https://deepwiki.com/ClickHouse/ClickHouse>
