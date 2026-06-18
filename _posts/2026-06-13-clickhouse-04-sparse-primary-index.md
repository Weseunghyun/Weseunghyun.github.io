---
title: "ClickHouse 완전 정복 4장 — sparse primary index: 8천만 행에서 한 줌만 읽는 법"
date: 2026-06-13 23:55:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "핵심"]
---
3장에서 우리는 MergeTree가 데이터를 불변(immutable) part로 쪼개 저장하고,
ORDER BY로 정렬한 채 백그라운드에서 머지한다는 것을 봤다.
그런데 한 가지 질문이 남는다.
정렬은 해뒀다 치자.
그럼 수억 행 중에서 "UserID = 749927693인 행"을 어떻게 빨리 찾아낼까?

전통적인 데이터베이스(MySQL, PostgreSQL 같은 OLTP 계열)는 이걸 B-Tree 인덱스로 푼다.
모든 행, 혹은 거의 모든 키 값마다 인덱스 엔트리를 만들어 트리로 매단다.
한 행을 정확히 찍어 꺼내는 데는 이게 최고다.

ClickHouse는 다르게 푼다.
정반대 방향이라고 해도 좋다.
ClickHouse의 기본 키 인덱스는 **sparse(희소) 인덱스**다.
모든 행에 인덱스 엔트리를 다는 게 아니라,
**행 묶음 하나당 인덱스 엔트리 하나**만 둔다.

이 장에서는 그 "행 묶음 하나당 엔트리 하나"가 정확히 무엇인지,
그게 어떻게 8천만 행짜리 테이블에서 단 한 줌만 읽고 끝내는지를 끝까지 파본다.

---

## granule: ClickHouse가 읽는 가장 작은 단위

먼저 용어 하나.
ClickHouse는 정렬된 데이터를 **granule(그래뉼)**이라는 묶음으로 나눈다.
granule은 ClickHouse가 디스크에서 읽어 들이는 **가장 작은, 더는 쪼갤 수 없는 데이터 단위**다 (공식 문서 기준).

granule 하나에 들어가는 행 수는 `index_granularity` 설정으로 정해지고,
**기본값은 8192행**이다 (공식 문서 기준).
즉 데이터를 8192행씩 끊어 한 묶음으로 본다.

그리고 sparse 인덱스는 **각 granule의 첫 행의 기본 키 값**만 기록한다.
이 기록 하나를 **mark(마크)**라고 부른다.
공식 문서 표현 그대로 옮기면,
"part의 기본 인덱스는 granule(행 묶음)마다 인덱스 엔트리(즉 mark) 하나를 가진다."

비유를 하나 들자 — 이건 어디까지나 이해를 돕기 위한 비유다.
두꺼운 백과사전을 떠올려 보자.
B-Tree 방식은 책 뒤에 **모든 단어**의 색인을 다 만들어 두는 것이다.
sparse 방식은 그렇게 안 한다.
대신 **책을 8192쪽씩 끊어, 각 묶음의 첫 페이지에 적힌 단어**만 색인에 적어둔다.
"아, 750번 묶음은 'ㅅ'으로 시작하고 751번 묶음은 'ㅇ'으로 시작하네"
정도만 아는 것이다.

![](/assets/img/posts/clickhouse-04-sparse-primary-index/clickhouse-sparse-index.svg)
실제 mark가 가리키는 건 페이지가 아니라 압축 블록 오프셋이지만,
그 정밀한 메커니즘은 잠시 뒤에 본다.

왜 이렇게 듬성듬성 만들까?
이유는 단순하다.
인덱스가 작아야 **통째로 메모리에 올릴 수 있기** 때문이다.

---

## primary.idx와 .mrk: 인덱스가 작은 진짜 이유

ClickHouse는 mark들을 `primary.idx`라는 파일에 모아 둔다.
이 파일은 **N번째 행마다(N = index_granularity) 기본 키 값**을 저장하고,
**항상 메모리에 상주**한다 (공식 문서 기준).
최신 버전에서는 `PrimaryIndexCache`라는 메커니즘이 이 메모리 내 저장을 관리한다.

여기서 sparse 인덱스의 핵심 이득이 드러난다.
8192행당 엔트리가 하나면, 인덱스는 데이터의 약 8192분의 1 크기로 줄어든다.
공식 문서의 실측 예제를 보자.

| 항목 | 값 |
|---|---|
| 테이블 행 수 | 8,870,000행 (약 8.87M) |
| granule 개수 | 1,083개 |
| 압축 데이터 크기 | 206.94 MB |
| 비압축 데이터 크기 | 733.28 MB |
| primary index(`primary.idx`) 크기 | **96.93 KB** |

8.87M ÷ 8192 ≈ 1083.
8백만 행이 넘는 테이블의 인덱스가 고작 **96.93 KB**다 (공식 문서 기준).
이 정도면 메모리에 통째로 올려놓고 마음껏 뒤져도 부담이 없다.
B-Tree로 모든 행을 색인했다면 인덱스만으로도 수백 MB가 됐을 것이다.

그런데 인덱스가 granule의 "첫 행 키 값"만 안다면,
그 granule이 디스크 어디에 있는지는 어떻게 찾을까?
여기서 `.mrk`(mark) 파일이 등장한다.

`.mrk` 파일은 granule마다 **8바이트 오프셋 두 개의 쌍**을 저장한다 (공식 문서·소스 기준).

1. 압축 파일 안에서, 해당 데이터가 들어 있는 **압축 블록의 시작 오프셋**
2. 그 블록을 **압축 해제했을 때, 데이터가 시작되는 위치 오프셋**

왜 오프셋이 두 개일까?
ClickHouse는 데이터를 압축 블록 단위로 디스크에 쓴다.
첫 번째 오프셋으로 "어느 압축 블록을 디스크에서 읽어올지"를 정하고,
그 블록을 메모리에서 압축 해제한 뒤,
두 번째 오프셋으로 "그 안에서 어디부터 우리 데이터인지"를 짚는다.

보통은 압축 블록이 mark 경계에 맞춰 정렬돼 있어서, 두 번째 오프셋이 0이다.
하지만 작은 granule 여러 개가 한 압축 블록 안에 들어가는 경우,
두 번째 오프셋이 0이 아니게 된다 (공식 문서·소스 기준).
ClickHouse 내부에서는 `MergeTreeMarksLoader`라는 컴포넌트가 이 mark 파일을 읽어 들인다.

정리하면 이렇다.
`primary.idx`는 "키 값 → 몇 번째 granule?"을 알려주고,
`.mrk`는 "그 granule → 디스크의 정확한 어느 바이트?"를 알려준다.
이 둘이 짝을 이뤄 sparse 인덱스를 완성한다.

---

## 쿼리가 도는 2단계: 이진 탐색 → granule만 읽기

이제 실제 쿼리를 생각해 보자.
`WHERE UserID = 749927693` 같은 조건이 들어오면 ClickHouse는 **2단계**로 움직인다 (공식 문서·소스 기준).

**1단계 — 후보 granule 고르기.**
메모리에 상주하는 `primary.idx`의 mark들을 **이진 탐색(binary search)**한다.
mark는 각 granule의 첫 키 값이고, 데이터는 정렬돼 있으므로,
조건에 맞을 가능성이 있는 granule의 범위(`MarkRanges`)를 빠르게 좁힐 수 있다.

**2단계 — 고른 granule만 읽기.**
1단계에서 선택된 granule만 `.mrk` 오프셋을 따라 디스크에서 읽어 온다.
나머지 granule은 아예 손도 대지 않는다.
전체 스캔(full scan)을 피하는 것이다.

내부적으로는 WHERE 절이 `KeyCondition`이라는 구조로 분석되어
RPN(역폴란드 표기법) 형태로 변환되고,
이 RPN으로 mark들을 이진 탐색해 후보 `MarkRanges`를 결정한다 (소스 기준).

공식 문서의 같은 예제로 숫자를 보자.

| 단계 | 수치 |
|---|---|
| 전체 granule(=mark) 수 | 1,083개 |
| 기본 키 첫 컬럼(UserID) 조회 시 이진 탐색 | **19 스텝** (공식 문서 기준) |
| 실제로 읽는 데이터 | granule 1개 = 8,192행 |
| 안 읽고 건너뛴 데이터 | 나머지 약 8.86M행 |

1083개를 이진 탐색해 일치하는 granule을 찾는 데 공식 문서 기준 19스텝이 걸리고,
그 결과 8.87M 전체가 아니라 **단 8192행 한 granule만** 읽는다.
(엄밀히 단순 log₂(1083)은 약 10인데 문서 수치는 19스텝이다.
범위 경계 처리 등 내부 탐색 구현 때문으로 보이며,
정확한 이유까지 단정하기보다 "문서 기준 19스텝"으로 받아들이면 된다.)

여기서 sparse 인덱스의 트레이드오프가 분명해진다.
B-Tree처럼 정확히 그 행 하나만 집어내지는 못한다 — 항상 granule 단위(최소 8192행)로 읽는다.
대신 인덱스가 작아 메모리에 다 올라가고,
대량 분석 쿼리에서 읽을 데이터를 granule 단위로 통째 쳐내는 데 압도적으로 유리하다.
이게 OLTP의 "한 행 콕 집기"가 아니라 OLAP의 "범위 싹 훑기"에 맞춰진 설계다.
1장에서 ClickHouse가 point lookup에 약하다고 했던 이유의 절반이 바로 여기에 있다.

---

## 복합 키의 순서가 모든 것을 바꾼다

ORDER BY에 컬럼을 여러 개 쓰는 복합 키일 때,
**컬럼을 어떤 순서로 놓느냐**가 성능을 크게 좌우한다.
5장에서 다룰 스키마 설계와도 직결되는, 실무에서 가장 자주 틀리는 지점이다.

이유는 sparse 인덱스가 **첫 컬럼 기준으로만** 깔끔한 이진 탐색을 할 수 있기 때문이다.
첫 컬럼이 아닌 보조 키 컬럼으로 필터링하면,
연속된 정확한 키 범위를 잡을 수 없어서
**generic exclusion search**라는 다른 알고리즘이 동작한다 (공식 문서·소스 기준).

이건 이진 범위 탐색처럼 한 방에 범위를 자르는 게 아니라,
조건에 안 맞는 mark 구간을 반복적으로 더 잘게 쪼개고 배제해 나가며 좁히는 방식이다.
mark 세그먼트를 검사해서 조건에 안 맞으면 더 작게 분할해 다시 평가하고,
개별 mark가 후보로 추가되거나 폐기될 때까지 반복한다 (소스 기준).
그래서 효율이 **선행 컬럼의 cardinality(고유값 다양성)**에 크게 좌우된다.

공식 문서의 실측 예제가 이걸 극적으로 보여준다.
같은 `UserID` 필터인데 ORDER BY 컬럼 순서만 다르다.

| ORDER BY | UserID 필터 시 처리한 행 |
|---|---|
| `(URL, UserID, IsRobot)` | **7,920,000행** (약 7.92M) |
| `(IsRobot, UserID, URL)` | **20,320,000... 가 아니라 20,320행** (약 20.32K) |

URL을 앞에 둔 경우 UserID 필터로는 거의 못 걸러서 7.92M행을 처리한다.
반면 cardinality가 낮은 IsRobot을 앞에 두니, 같은 필터가 20.32K행만 처리하고 끝난다.
약 390배 차이다.

여기서 ClickHouse 설계의 경험칙이 나온다.
**ORDER BY는 보통 카디널리티가 낮은 컬럼을 앞에 둔다** (공식 문서 기준).
낮은 cardinality 컬럼이 앞에 있어야 같은 값이 길게 이어지고,
generic exclusion search가 큰 덩어리를 한 번에 쳐낼 수 있기 때문이다.

---

## 정렬 순서의 두 번째 효과: 압축률

ORDER BY 컬럼 순서는 인덱스 효율만 바꾸는 게 아니다.
**압축률**까지 직접 바꾼다.
2장에서 본 컬럼 기반 압축이 여기서 정렬과 만난다.

원리는 단순하다.
압축은 인접한 값이 서로 비슷할수록 잘 된다.
데이터가 어떤 컬럼 기준으로 잘 정렬돼 있으면, 그 컬럼의 인접 값들이 비슷해진다.

같은 `UserID` 컬럼에 대한 공식 문서 예제를 보자.

| 정렬 상태 | UserID 컬럼 압축 결과 |
|---|---|
| 해당 컬럼 기준으로 잘 정렬됨 | **39:1** (877.47 KiB) |
| 잘 정렬 안 됨 | **3:1** (11.24 MiB) |

같은 컬럼인데 정렬이 잘 됐을 때는 39:1, 안 됐을 때는 3:1로 압축됐다 (공식 문서 기준).
디스크 사용량이 10배 넘게 차이 난다.
즉 ORDER BY 컬럼 순서는 **인덱스 스킵 효율 + 압축률**이라는 이중 효과를 갖는다.
설계 단계에서 한 번 정해 두면 두고두고 영향을 미치는 결정이라는 뜻이다.

---

## PRIMARY KEY와 ORDER BY를 분리하기

지금까지는 정렬 키(ORDER BY)와 기본 키(PRIMARY KEY)를 같은 것처럼 다뤘다.
3장에서 본 대로 보통은 둘이 같다.
하지만 둘을 **분리**할 수도 있다.
그럴 때 지켜야 할 규칙이 하나 있다.

공식 문서 인용:
"둘 다 지정한다면, 기본 키는 정렬 키의 prefix(앞부분)여야 한다."
즉 PRIMARY KEY는 ORDER BY 컬럼의 **앞쪽 일부**여야 한다.

왜 이런 게 필요할까?
앞에서 봤듯 `primary.idx`는 메모리에 상주한다.
정렬은 더 많은 컬럼으로 촘촘하게 하고 싶지만,
메모리에 올릴 인덱스는 가볍게 유지하고 싶을 때 이 분리가 유용하다.

```sql
-- PRIMARY KEY(메모리 상주 인덱스)는 앞쪽 2컬럼,
-- 정렬은 3컬럼으로 더 촘촘하게.
-- PRIMARY KEY는 ORDER BY의 prefix여야 함.
CREATE TABLE hits
(
    is_robot UInt8,
    user_id  UInt64,
    url      String
)
ENGINE = MergeTree
PRIMARY KEY (is_robot, user_id)
ORDER BY (is_robot, user_id, url);
```

이렇게 하면 디스크 정렬은 `(is_robot, user_id, url)` 세 컬럼 기준으로 이뤄지지만,
메모리에 올라가는 인덱스는 `(is_robot, user_id)` 두 컬럼 값만으로 작게 유지된다.

---

## data-skipping 인덱스: 보조 인덱스라는 안전장치

지금까지 본 sparse primary index는 **기본 키**에만 작동한다.
그럼 기본 키가 아닌 컬럼으로 필터링할 때는?
이때 쓰는 게 **data-skipping(데이터 스킵) 인덱스**, 즉 보조 인덱스다.

이름 그대로, WHERE 조건과 도저히 맞을 수 없는 granule을 **건너뛰게** 해주는 인덱스다.
단, 두 가지를 분명히 알아야 한다 (공식 문서 기준).

첫째, 이 인덱스는 **MergeTree 계열에서만** 동작한다.
둘째, 그리고 더 중요한데 — **그 자체로는 읽기를 빠르게 하지 않는다.**
인덱스 표현식과 실제 데이터(주로 기본 키) 사이에 **강한 상관관계**가 있을 때만 효과가 있다.
상관관계가 없으면 granule을 거의 못 건너뛰면서 자원만 쓴다.
그래서 만들고 나면 반드시 `EXPLAIN indexes = 1`로 실제로 스킵하는지 확인해야 한다.

data-skipping 인덱스에는 `GRANULARITY n`이라는 파라미터가 붙는다.
이건 인덱스 블록 하나가 기본 데이터 granule을 **몇 개씩 묶는지**를 정한다 (공식 문서 기준).
8192행 granule에서 `GRANULARITY 4`라면, 32,768행(8192 × 4)이 인덱스 한 블록이 된다.
`GRANULARITY 1`은 granule마다, `GRANULARITY 10`은 10 granule마다 인덱스 블록 하나다.

주요 인덱스 타입을 정리하면 이렇다 (공식 문서 기준).

| 타입 | 저장하는 것 | 잘 맞는 경우 |
|---|---|---|
| `minmax` | 블록당 최소/최대값 | 정렬과 상관된 컬럼(시간·증가 ID 등)의 범위 쿼리. 비용이 가장 저렴한 편 |
| `set(max_rows)` | 블록당 고유값(최대 max_rows개, 0=무제한) | granule 내 고유값은 적은데 전체로는 다양한 컬럼의 동등 비교 |
| `bloom_filter([fpr])` | 블룸 필터로 집합 멤버십 | 고-cardinality 동등/IN 조건, 배열·맵. fpr 기본 0.025 |
| `ngrambf_v1(...)` | n-gram을 블룸 필터에 | 부분 문자열/LIKE, 단어 경계 없는 언어 |
| `tokenbf_v1(...)` | 토큰(단어)을 블룸 필터에 | 로그 등 전체 단어 매칭 |

각 타입을 조금 더 풀면 —
`minmax`는 블록의 최소·최대만 들고 있다가, 찾는 값이 그 범위 밖이면 통째로 건너뛴다.
시간이나 증가하는 ID처럼 정렬 순서대로 값이 천천히 변하는(loosely sorted) 컬럼에 잘 맞는다.
`set`은 블록당 고유값을 한도까지 모아 둬서, 그 안에 없으면 건너뛴다.
`bloom_filter`는 문자열을 그대로(as is) 저장해 정확 일치·`has()` 같은 데 쓴다 — 부분 문자열(LIKE)에는 부적합하다.

문자열 부분 검색용인 `ngrambf_v1`과 `tokenbf_v1`은 공식 MergeTree 문서의 파라미터 순서가 정본이다.

- `ngrambf_v1(n, size_of_bloom_filter_in_bytes, number_of_hash_functions, random_seed)` — 문자열을 n-gram(보통 3~4글자)으로 쪼개 저장.
  임의의 부분 문자열 검색에.
- `tokenbf_v1(size_of_bloom_filter_in_bytes, number_of_hash_functions, random_seed)` — 비-영숫자 경계로 단어를 잘라 저장.
  전체 단어 매칭에.
  n-gram 크기만 없을 뿐 나머지 파라미터는 같다.

> 🔴 중요(최신성, 2026-06-13 기준): `ngrambf_v1`과 `tokenbf_v1`은 현재 **권장되지 않는다(deprecated)**.
> ClickHouse 26.2부터 GA된 **`text` 인덱스(full-text index)**가 전체 텍스트 검색의 권장 방식이다 (공식 문서 기준).
> 오늘 기준 stable은 26.4, LTS는 26.3이므로, 새로 만든다면 `text` 인덱스를 먼저 검토하는 게 맞다.
> (`text` 인덱스의 상세 사용법은 이 장 범위 밖이라 깊이 다루지 않는다.)

아래는 data-skipping 인덱스를 생성·추가하고 실제 스킵을 검증하는 DDL이다.

```sql
CREATE TABLE events
(
    ts        DateTime,
    user_id   UInt64,
    url       String,
    message   String,
    INDEX idx_url url TYPE bloom_filter(0.01) GRANULARITY 4,
    INDEX idx_msg message TYPE tokenbf_v1(8192, 3, 0) GRANULARITY 4
)
ENGINE = MergeTree
ORDER BY (ts, user_id)
SETTINGS index_granularity = 8192;

-- 기존 테이블에 추가 (기존 파트는 MATERIALIZE 필요)
ALTER TABLE events ADD INDEX idx_ts_minmax ts TYPE minmax GRANULARITY 1;
ALTER TABLE events MATERIALIZE INDEX idx_ts_minmax;

-- 실제로 granule을 스킵하는지 검증
EXPLAIN indexes = 1
SELECT count() FROM events WHERE url = 'https://example.com';
```

(위 예제는 파라미터 순서를 공식 문서대로 적었을 뿐, 직접 실행해 검증한 결과는 아니다.
실무에선 `text` 인덱스 쪽을 먼저 고려하자.)

---

## 인덱스를 내 눈으로 들여다보기

마지막으로, 지금까지 설명한 인덱스 구조를 직접 확인해 보는 방법이다.
ClickHouse는 자기 내부 상태를 system 테이블로 노출한다 (13장에서 본격적으로 다룬다).

```sql
SELECT
    table, name, rows, marks,
    formatReadableSize(primary_key_bytes_in_memory) AS pk_in_mem
FROM system.parts
WHERE table = 'hits' AND active;

-- mergeTreeIndex 테이블 함수로 인덱스/마크 파일 내용 직접 조회
SELECT * FROM mergeTreeIndex('default', 'hits') LIMIT 10;
```

`system.parts`의 `marks` 컬럼이 곧 granule 개수이고,
`primary_key_bytes_in_memory`가 그 part의 인덱스가 메모리에서 차지하는 크기다.
앞에서 본 "1083 granule, 96.93 KB"가 바로 이런 컬럼으로 확인되는 값이다.

---

## 왜 이렇게 설계했을까

다시 처음 질문으로 돌아가자.
ClickHouse는 왜 B-Tree 대신 sparse 인덱스를 골랐을까?

OLTP는 "1번 주문 행을 정확히 꺼내라" 같은 point lookup이 주력이다.
그래서 모든 행을 색인하는 B-Tree가 맞다.

OLAP은 다르다.
"지난 한 달, 봇이 아닌 사용자의 페이지뷰를 집계하라" 같은 **대량 범위 스캔**이 주력이다.
이럴 땐 행 하나를 정밀하게 찾는 능력보다,
**읽을 필요 없는 거대한 덩어리를 통째로 건너뛰는 능력**이 훨씬 중요하다.
sparse 인덱스는 정확히 그걸 위해, 인덱스를 작게 만들어 메모리에 통째로 올리고,
granule 단위로 데이터를 쳐낸다.

그래서 ClickHouse에서 ORDER BY 설계는 단순한 정렬 지정이 아니다.
인덱스 효율, 압축률, 그리고 쿼리가 읽을 데이터 양을 한꺼번에 결정하는, 가장 중요한 한 줄이다.

다음 **5장**에서는 이 정렬 키에 올릴 컬럼들 — 즉 데이터 타입과 스키마 설계를 다룬다.
`LowCardinality`나 적절한 정수 타입 선택이 압축률과 인덱스에 어떻게 더 작용하는지,
방금 본 원리 위에서 이어 본다.

---

## 참고

- ClickHouse 공식 문서 — A Practical Introduction to Sparse Primary Indexes: https://clickhouse.com/docs/optimize/sparse-primary-indexes
- ClickHouse 공식 문서 — Understanding Data Skipping Indexes: https://clickhouse.com/docs/optimize/skipping-indexes
- ClickHouse 공식 문서 — MergeTree 엔진 (Available Types of Indices): https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree
- ClickHouse 소스 코드 분석 (DeepWiki): https://deepwiki.com/ClickHouse/ClickHouse
