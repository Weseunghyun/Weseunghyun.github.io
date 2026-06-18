---
title: "ClickHouse 완전 정복 7장 — Projection: 같은 테이블 안에 정렬·집계가 다른 또 하나의 사본을 둔다"
date: 2026-06-13 23:52:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "심화"]
---
4장에서 sparse primary index를 봤다.
ClickHouse는 `ORDER BY`로 정한 정렬키 순서로 데이터를 쌓고, 그 순서를 따라 granule을 건너뛰며 읽는다.
그래서 정렬키(=기본키) 앞쪽 컬럼으로 거르는 쿼리는 빠르다.
하지만 정렬키에 없는 컬럼으로 거르면 사정이 다르다.
인덱스가 도와주지 못하니, 사실상 그 컬럼 전체를 훑어야 한다.

테이블 하나에 정렬키는 하나뿐이다.
그런데 현실의 분석 쿼리는 거르는 기준이 여러 개다.
어떤 화면은 `user_id`로 거르고, 어떤 리포트는 `event_date`로 거르고, 또 어떤 대시보드는 `region`별로 집계한다.
정렬키를 셋 다 동시에 만족시킬 수는 없다.

6장에서 본 Materialized View가 이 문제에 대한 한 가지 답이었다 — 삽입 시점에 미리 계산해 별도 타깃 테이블에 쌓아 둔다.
7장에서 볼 **Projection**은 다른 답이다.
한 줄로 핵심을 던지면 이렇다.

> Projection은 같은 테이블 안에, 정렬 순서나 집계 방식이 다른 **또 하나의 데이터 사본**을 두고, 쿼리가 들어오면 옵티마이저가 둘 중 더 적게 읽는 쪽을 알아서 고르게 하는 장치다.

공식 문서는 Projection을 두 가지 용도로 설명한다.
하나는 정렬키가 아닌 컬럼으로 거를 때의 필터링 가속이고,
다른 하나는 사전 집계(pre-aggregation)다(공식 문서 기준).

![](/assets/img/posts/clickhouse-07-projection/clickhouse-projection.svg)

## 비유: 같은 책의 색인을 두 벌 끼워 둔다

이건 이해를 돕는 비유다 — 실제 구조는 바로 아래 메커니즘에서 다시 정확히 짚는다.

두꺼운 전화번호부를 떠올려 보자.
이 책은 "성(姓) 순서"로 정렬돼 있다.
성을 알면 금방 찾지만, "전화번호 뒷자리가 1234인 사람"을 찾으려면 책을 처음부터 끝까지 넘겨야 한다.

Projection은 이 책 뒤에, 같은 내용을 "전화번호 순서로 다시 정렬한" 사본을 한 벌 더 끼워 두는 것과 같다.
이제 전화번호로 찾을 땐 그 사본을 펴면 된다.
중요한 건, 두 사본이 **같은 책의 일부**라는 점이다.
원본에 새 사람을 추가하면 사본도 같이 갱신된다 — 따로 관리할 별책이 아니다.

이 "같은 책의 일부"라는 성질이, 뒤에서 볼 Materialized View와의 결정적 차이다.

## 메커니즘 1 — Projection은 part 안에 사는 익명 MergeTree다

3장에서 MergeTree의 데이터는 part라는 불변 덩어리로 저장된다고 했다.
Projection을 정의하면, ClickHouse는 **각 부모 data part 안에** 별도의 서브디렉터리를 만들고 거기에 Projection 데이터를 또 하나의 작은 MergeTree 파트로 저장한다(공식 문서·소스 기준).
즉 Projection은 테이블 바깥에 따로 떠 있는 객체가 아니라, 부모 part에 딸려 사는 익명의 내부 MergeTree다.

이 내부 파트의 성격은 Projection 종류에 따라 다르다.

- **normal projection**: 부모와 같은 데이터를, 하지만 **다른 `ORDER BY`**로 정렬해 들고 있다.
  전화번호부 비유에서 "전화번호 순으로 다시 정렬한 사본"이 이쪽이다.
- **aggregate projection**: `GROUP BY`로 미리 집계한 결과를 들고 있다.
  이쪽은 내부적으로 AggregatingMergeTree로 저장된다(공개적으로 확인되는 범위에선 그렇다).

문법은 `ALTER TABLE`로 Projection을 테이블에 붙이는 형태다.

```sql
-- normal projection: 정렬키가 아닌 user_id 로 거르는 쿼리 가속
ALTER TABLE events
ADD PROJECTION proj_by_user
(
    SELECT *
    ORDER BY user_id
);

-- aggregate projection: region 별 집계를 미리 계산
ALTER TABLE events
ADD PROJECTION proj_region_agg
(
    SELECT
        region,
        count(),
        sum(payload_bytes)
    GROUP BY region
);
```

여기서 한 가지 짚어 둘 것.
Projection은 부모 part의 일부로 같이 저장되므로, 부모 테이블에 데이터가 들어오면 Projection도 **자동으로 함께 유지된다.**
6장의 증분 MV가 "삽입 트리거"로 별도 타깃에 결과를 흘려보내던 것과 달리,
Projection은 트리거가 아니라 part 자체의 구성 요소다.

## 메커니즘 2 — 옵티마이저가 part마다 알아서 고른다

Projection을 만들었다고 해서 쿼리를 다시 쓸 필요는 없다.
사용자는 원래대로 원본 테이블에 쿼리를 던진다.
그러면 옵티마이저가 쿼리를 보고, **part마다** 원본을 읽는 것과 Projection을 읽는 것 중 스캔량이 더 적은 쪽을 자동으로 고른다(공식 문서 기준).

"part마다"라는 점이 중요하다.
어떤 part에는 해당 Projection이 채워져 있고 어떤 part에는 없을 수 있다(뒤의 MATERIALIZE 절 참고).
Projection이 없는 part는 그냥 원본을 읽는 fallback으로 처리된다.
그래서 Projection을 막 추가한 직후엔, 기존 part는 원본을 읽고 새로 들어오는 part만 Projection의 혜택을 받는 식이 된다.

옵티마이저가 실제로 Projection을 썼는지는 `system.query_log`의 `projections` 항목으로 확인할 수 있다(공식 문서 기준).
13장 운영·모니터링에서 system 테이블을 더 다룬다.

```sql
SELECT query, projections
FROM system.query_log
WHERE type = 'QueryFinish'
ORDER BY event_time DESC
LIMIT 5;
```

이 "투명하게 알아서 고른다"는 점이 Projection의 가장 큰 매력이다.
애플리케이션 쿼리를 한 줄도 바꾸지 않고, 정렬키가 다른 접근 패턴까지 가속할 수 있다.

## 메커니즘 3 — ADD는 정의만, MATERIALIZE가 과거 part를 채운다

6장의 MV가 "만든 이후의 INSERT만 본다"는 함정을 가졌듯,
Projection에도 비슷한 결의 구분이 있다.
`ALTER TABLE ... ADD PROJECTION`은 **정의만 등록**할 뿐, 이미 존재하는 part에는 Projection 데이터를 채우지 않는다(공식 문서 기준).

기존 part까지 Projection을 채우려면 `MATERIALIZE`를 따로 실행해야 한다.
이건 기존 데이터를 다시 읽어 Projection 파트를 만드는 작업이라 mutation으로 동작한다 — 무거울 수 있다.

```sql
-- 1) 정의만 등록 (기존 part 는 아직 비어 있음)
ALTER TABLE events ADD PROJECTION proj_by_user (SELECT * ORDER BY user_id);

-- 2) 기존 part 까지 채움 (mutation — 무거움)
ALTER TABLE events MATERIALIZE PROJECTION proj_by_user;
```

반대로 지우는 쪽도 두 갈래다.

- `DROP PROJECTION`: 정의와 파일을 모두 제거한다.
- `CLEAR PROJECTION`: 정의는 남기고 디스크의 Projection 파일만 비운다.

| 명령 | 정의 | 디스크 파일 | 용도 |
|---|---|---|---|
| `ADD PROJECTION` | 등록 | 안 채움(신규 part부터) | 새 Projection 선언 |
| `MATERIALIZE PROJECTION` | 유지 | 기존 part까지 채움(mutation) | 과거 데이터 백필 |
| `CLEAR PROJECTION` | 유지 | 비움 | 정의는 두고 공간만 회수 |
| `DROP PROJECTION` | 제거 | 제거 | 완전 삭제 |

## 메커니즘 4 — Projection vs Materialized View, 무엇이 다른가

여기서 6장의 MV와 정면으로 비교해 두자.
둘 다 "미리 계산해 둔다"는 점은 같지만, **결과를 어디에 두느냐**가 다르고 그 차이가 일관성으로 이어진다.

MV는 결과를 **별도의 타깃 테이블**에 쌓는다.
타깃 테이블은 소스와 독립된 객체라, 소스와 타깃이 항상 같은 시점을 보장하지는 않는다(atomic 일관성 보장 없음).
반면 Projection은 부모 part의 일부라, base 데이터와 Projection이 **atomic하게 일관**되고 자동으로 함께 유지된다(공식 문서 기준).
원본에 들어간 데이터는 같은 part 안의 Projection에도 같이 반영되므로, "둘이 어긋난 시점"이 원리상 생기지 않는다.

| 항목 | Projection | Materialized View |
|---|---|---|
| 저장 위치 | 부모 part 안(같은 테이블) | 별도 타깃 테이블 |
| base와 일관성 | atomic·자동 유지 | atomic 보장 없음(별도 객체) |
| 계산 시점 | part에 함께 저장 | 삽입 트리거로 타깃에 적재 |
| 쿼리 변경 | 불필요(투명, 옵티마이저가 선택) | 보통 타깃 테이블을 직접 조회 |
| JOIN/팬아웃 | 불가(아래 제약 참고) | 가능(JOIN·다른 스키마·fan-out) |

선택의 기준은 비교적 또렷하다.
**단일 테이블을 쿼리 변경 없이 투명하게 가속**하고 싶으면 Projection이고,
**JOIN이 필요하거나, 다른 스키마로 변형하거나, 한 소스에서 여러 타깃으로 fan-out**해야 하면 MV다(공식 문서 기준).

## 메커니즘 5 — Projection의 제약들

Projection은 강력하지만 못 하는 것이 분명하다.
이걸 알아야 "왜 내 쿼리는 Projection을 안 타지?"에서 헤매지 않는다.

공식 문서가 드는 제약은 대략 이렇다(공식 문서 기준).

- **JOIN, WHERE 절, 그리고 Projection 체이닝(projection 위에 projection)을 쓸 수 없다.**
  Projection 정의 안에서는 단일 테이블의 `SELECT`/`ORDER BY`/`GROUP BY` 정도만 표현된다.
- **부모 테이블과 다른 TTL을 가질 수 없고, lightweight DELETE/UPDATE 같은 lightweight DML과 함께 쓸 수 없다.**
- **`ALIAS` 컬럼으로 `ORDER BY` 할 수 없다.**
- **다른 정렬키를 쓰는 normal projection은 데이터를 중복 저장한다.**
  전화번호부를 한 벌 더 끼우는 셈이니 당연하다 — 정렬 순서가 다르면 같은 데이터가 두 번(이상) 디스크에 놓인다.

마지막 항목이 Projection의 본질적 비용이다.
필터 가속을 얻는 대가로 디스크 사용량과 삽입·머지 부담이 늘어난다.
그래서 "아무 컬럼에나 일단 Projection을 건다"는 접근은 좋지 않다.
정말 자주 쓰는, 정렬키로 못 거르는 접근 패턴에만 선별적으로 거는 게 맞다.

## 최신 동향 — _part_offset로 중복 저장 줄이기

방금 본 "중복 저장" 비용을, 비교적 최근 버전이 한 가지 방식으로 줄여 준다.
공개적으로 확인되는 범위에선, v25.5부터 normal projection이 데이터 전체를 복제하는 대신 **정렬키와 `_part_offset`(부모 part 안에서의 행 위치)만 저장**할 수 있다(공식 문서 기준).

원리는 이렇다.
Projection에는 "어떻게 정렬됐는지"와 "원본의 몇 번째 행인지"만 들고 있다가,
쿼리가 들어오면 그 오프셋을 따라 **base 데이터를 읽어** 나머지 컬럼을 가져온다.
정렬 정보만 별도로 들고 있으니 인덱스 역할은 하면서, 전체 컬럼을 두 벌 저장하는 오버헤드는 피한다.

다만 이 방식도 공짜는 아니다.
쿼리 시점에 base part를 한 번 더 읽어야 하니, 완전 복제 방식보다 그 부분에선 손해를 본다.
"디스크 절감 vs 쿼리 시 base 재읽기"의 trade-off라고 보면 된다.
버전에 따라 동작이 다를 수 있으니, 쓰기 전에 자기 버전(2026-06 기준 stable 26.4 · LTS 26.3)에서 실제 거동을 확인하는 편이 안전하다.

## "왜 이렇게?" — 같은 테이블 안에 사본을 두는 이유

마지막으로 한 번 더 질문해 보자.
왜 ClickHouse는 MV처럼 별도 테이블에 결과를 쌓지 않고, 굳이 part 안에 사본을 욱여넣는 모델을 따로 두었을까?

핵심은 일관성과 투명성이다.
별도 테이블 방식(MV)은 유연하지만, 소스와 타깃이 어긋날 수 있고 사용자가 "어떤 쿼리는 타깃을 봐라"라고 알고 있어야 한다.
반면 Projection을 part의 일부로 묶으면, base와의 일관성이 part 단위로 저절로 보장되고,
사용자는 늘 원본 테이블에만 쿼리하면 옵티마이저가 알아서 빠른 사본을 고른다.
즉 "쿼리를 바꾸지 않고, 데이터가 어긋날 걱정 없이" 두 번째 접근 패턴을 얻는 것 — 이게 Projection이 노리는 자리다.

그 대가가 이번 장에서 본 제약들(JOIN·체이닝 불가, 중복 저장, MATERIALIZE의 무게)이다.
이 대가가 받아들이기 어려운 경우 — JOIN이나 다른 스키마로의 변형이 필요한 경우 — 를 위해 6장의 MV가 따로 존재한다.
ClickHouse는 두 도구를 우열 관계가 아니라 **역할 분담**으로 둔다.
단일 테이블 투명 가속은 Projection, 변형·JOIN·fan-out은 MV다.

다음 8장에서는 데이터 안전과 가용성으로 넘어간다 — ReplicatedMergeTree와 Keeper로 같은 데이터를 여러 노드에 복제하는 구조를 본다.
지금까지의 part·정렬·사본 이야기가 "한 노드 안"의 이야기였다면, 8장부터는 그 part들을 노드 사이에서 어떻게 똑같이 유지하느냐의 문제로 넘어간다.

## 참고

- ClickHouse Docs — Projections (데이터 모델링): https://clickhouse.com/docs/data-modeling/projections
- ClickHouse Docs — ALTER TABLE ... PROJECTION: https://clickhouse.com/docs/sql-reference/statements/alter/projection
- ClickHouse Docs — Materialized Views versus Projections: https://clickhouse.com/docs/managing-data/materialized-views-versus-projections
- ClickHouse 소스/아키텍처(부모 part 내 .proj 저장 구조): https://deepwiki.com/ClickHouse/ClickHouse
