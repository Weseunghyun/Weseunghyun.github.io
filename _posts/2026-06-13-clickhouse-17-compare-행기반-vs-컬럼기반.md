---
title: "ClickHouse 완전 정복 17장 — 비교: 행기반(RDB/OLTP)과 컬럼기반(ClickHouse/OLAP), 갈라진 두 갈래의 기원"
date: 2026-06-13 23:42:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "비교"]
---
이 시리즈를 여기까지 따라왔다면 ClickHouse가 "왜 빠른가"(2장)도, MergeTree가 데이터를 어떻게 쌓는가(3장)도, sparse primary index가 어떻게 granule을 건너뛰는가(4장)도 이미 봤다.
이번 장은 한 걸음 물러나 더 근본적인 질문을 던진다.

우리가 흔히 "데이터베이스"라고 부르는 PostgreSQL·MySQL·Oracle과, ClickHouse는 같은 SQL을 쓰고 같은 테이블 모양을 보여준다.
그런데 왜 하나는 결제 시스템에 들어가고, 다른 하나는 로그 분석에 들어가는가.
둘은 어디서 갈라졌고, 그 갈라짐의 뿌리에는 무슨 결정이 있었는가.

답은 단 한 가지 — **데이터를 디스크에 행(row)으로 줄세우느냐, 컬럼(column)으로 줄세우느냐** — 에 거의 다 들어 있다.
이 한 줄짜리 결정이 50년에 걸쳐 두 개의 데이터베이스 세계를 만들었다.

---

## 30초 요약

- **행기반(row-oriented)**: 한 행의 모든 컬럼 값을 디스크에 연속으로 저장한다.
  한 레코드를 통째로 읽고 쓰는 데 유리 → **OLTP**(결제·주문·계정 같은 트랜잭션 처리).
- **컬럼기반(column-oriented)**: 같은 컬럼의 값들을 한데 모아 저장한다.
  특정 컬럼만 골라 대량으로 스캔·집계하는 데 유리 → **OLAP**(분석·집계·리포팅).
- 관계형 모델은 1970년(Codd), OLAP이라는 용어는 1993년(Codd 백서)에 나왔다.
  컬럼 저장 아이디어는 1985년 DSM 논문, 2005년 C-Store 논문으로 학술적 분수령을 맞고 Vertica로 상업화됐다.
- ClickHouse는 2009년 Yandex에서 시작돼 2012년 Yandex.Metrica 프로덕션에 투입, 2016년 6월 오픈소스가 됐다 (공식 문서 기준).
- "둘 중 뭐가 좋냐"는 질문은 틀렸다.
  **워크로드가 행을 통째로 만지느냐, 컬럼을 가로질러 집계하느냐**에 따라 답이 갈린다.

---

## 1. 먼저 기원: 행기반은 "거래"에서, 컬럼기반은 "분석"에서 자랐다

### 행기반의 뿌리 — 관계형 모델과 트랜잭션

우리가 아는 거의 모든 전통 RDB의 이론적 토대는 1970년 E.F.
Codd(IBM)의 논문 'A Relational Model of Data for Large Shared Data Banks'에서 나왔다.
이 관계형 모델이 Oracle·DB2·PostgreSQL·MySQL로 이어졌다 (공개 정리 기준).

이들이 디스크에 데이터를 쌓는 방식은 단순하다.
**한 행의 모든 컬럼 값을 디스크에 연속으로, 그리고 행들을 차례차례(one after the other) 쌓는다** — 이것이 ClickHouse 공식 문서가 행기반을 설명하며 쓴 표현이다.

이 구조는 "한 레코드를 통째로 다루는 일"에 자연스럽다.
주문 한 건을 넣고, 계정 하나의 잔액을 고치고, 사용자 한 명을 PK로 찾아 전부 읽는 작업.
이런 작업을 **OLTP(Online Transaction Processing, 온라인 트랜잭션 처리)** 라고 부른다.

### 컬럼기반의 뿌리 — "읽기 위해 태어난" 저장 방식

반대편 갈래는 "쓰기"가 아니라 "읽기", 그중에서도 **분석을 위한 대량 읽기**를 위해 자랐다.

먼저 용어부터.
**'OLAP(Online Analytical Processing)'** 라는 말은 1993년 E.F.
Codd 등이 백서 'Providing OLAP to User-Analysts: An IT Mandate'에서 만들었다 (공개 정리 기준).
기존 OLTP를 비튼 명칭이고, Codd는 이 보고서에서 OLAP 도구가 갖춰야 할 12가지 규칙을 정의했다.
개념의 뿌리는 더 거슬러 1970년 다차원 DB 기술 Express(나중에 Oracle이 인수)까지 닿고, 1980년대부터 아이디어가 발전했다.

OLAP은 수백만~수십억 행을 스캔해서 합·평균·카운트로 집계하는 일이고, OLTP는 소수 행을 밀리초 단위로 읽고 쓰는 일이다.
이 둘은 데이터 모양은 같아도 **디스크를 두드리는 패턴이 정반대**다.

### 컬럼 저장의 학술적 계보

컬럼으로 저장한다는 발상 자체는 꽤 오래됐다.
개념적 기원으로 흔히 1985년 Copeland & Khoshafian의 'A Decomposition Storage Model(DSM)'(SIGMOD)이 꼽히고, 연구의 뿌리는 1970년대(예: 1969 TAXIR)까지 거슬러 올라간다 (공개 정리 기준).
이후 SybaseIQ, Wisconsin PAX, MonetDB(CWI) 같은 계열이 이어졌다.

상용 쪽에서는 **Sybase IQ가 최초의 상용 컬럼 지향 DB로 꼽힌다.**
1990년대 초 Expressway Technologies의 Expressway 103 엔진에서 출발해 Sybase가 인수한 뒤 1990년대 중반 'Sybase IQ'로 출시됐고, 이후 SAP IQ로 이어졌다.
다만 출처마다 최초 출시 연도가 미세하게 엇갈려서, 여기서는 "1990년대 중반"으로만 적어둔다.

학술적 분수령은 2005년 VLDB에서 발표된 **'C-Store: A Column-oriented DBMS'** 논문이다.
Michael Stonebraker·Daniel Abadi·Samuel Madden·Stanley Zdonik 등이 주도한 Brown·Brandeis·MIT·UMass Boston 공동 연구였다.
C-Store의 핵심 한 줄은 "쓰기가 아니라 읽기를 위해 DB를 최적화했다(optimizing the database for reading rather than writing)"이다.
이 연구 프로토타입은 Stonebraker 등이 창업한 **Vertica Analytic Database로 상업화**됐는데, 학술 시스템 연구가 곧장 상용 제품으로 이어진 대표 사례다 — 원본 오픈소스 프로젝트(마지막 릴리스 2006)보다 상용 fork가 오래 살아남았다.

---

## 2. ClickHouse는 어디서 왔나 — "비집계 원본 데이터로 실시간 리포트"

이 계보 위에 ClickHouse가 올라온다.

ClickHouse는 2009년 Yandex에서 Alexey Milovidov가 던진 질문에서 시작됐다.
**"비집계(non-aggregated) 원본 데이터로 실시간 분석 리포트를 만들 수 있는가?"** (공식 문서 기준)
3년 개발 끝에 2012년 세계 2위 웹 분석 플랫폼 Yandex.Metrica의 프로덕션에 투입됐고, 2016년 6월 15일 GitHub에서 Apache 2.0 라이선스로 오픈소스화됐다.
C++로 작성됐고 Linux/FreeBSD/macOS를 지원한다.

여기서 "비집계"라는 단어가 설계 철학의 핵심이다.
당시 대안은 데이터를 미리 집계해서 쌓는 방식이었는데, 그러면 두 가지 문제가 생긴다.
하나는 **미리 정해둔 집계 외의 커스텀 리포트를 못 만든다**는 것, 또 하나는 집계 차원 조합이 늘수록 미리 만들 표가 **조합 폭발(combinatorial explosion)** 한다는 것이다.
선행 시스템 OLAPServer는 숫자 타입만 지원했고, 실시간 증분 업데이트가 안 돼 하루 단위로 통째 재작성해야 했다 (공식 문서 기준).

규모를 보면 왜 이게 문제였는지 와닿는다.
2014년 4월 기준 Yandex.Metrica는 하루 약 120억 개의 이벤트(페이지뷰+클릭)를 추적했고, 수백 밀리초 안에 수백만 행을 스캔해야 했다 (공식 문서 기준).
이 "원본을 그대로 쌓되, 대량 스캔은 빨라야 한다"는 요구를 푸는 답이 바로 컬럼 저장이었다.

참고로 ClickHouse는 2021년 9월 미국 샌프란시스코에 ClickHouse Inc.로 법인화되며 상업화의 길로 들어섰고(Yandex로부터 스핀아웃), 이후 여러 차례 투자 라운드를 거쳐 성장했다 (공개 정리 기준).
Uber·Comcast·eBay·Cisco·CERN(LHCb) 등이 채택한 것으로 알려져 있는데, 각사가 구체적으로 "왜·어떻게" 적용했는지는 회사 기술블로그 1차 출처로 별도 검증이 필요한 부분이라 여기서는 채택 사실만 적어둔다.

---

## 3. 비유: 도서관의 책장과 색인 카드

저장 방식의 차이를 그림으로 잡아보자.
**이건 어디까지나 이해를 돕는 비유다 — 실제 구현은 2~4장에서 다룬 part·granule·압축 블록 구조다.**

학생 명부 테이블이 있다고 하자.
컬럼은 `이름·학번·학과·성적·등록일`.

- **행기반은 "책 한 권에 한 학생"** 처럼 쌓는다.
한 학생의 모든 정보(이름~등록일)가 한 권에 묶여 책장에 차례로 꽂힌다.
"12345번 학생 정보 다 줘"라고 하면 그 책 한 권만 꺼내면 끝 — 점조회가 빠르다.
대신 "전교생 성적 평균"을 구하려면 모든 책을 한 권씩 다 펴서 성적 페이지만 보고 다시 덮어야 한다.

- **컬럼기반은 "항목별 카드 묶음"** 처럼 쌓는다.
이름은 이름끼리, 성적은 성적끼리 묶음으로 따로 보관한다.
"전교생 성적 평균"은 성적 묶음 하나만 통째로 들고 와 더하면 끝 — 다른 묶음은 건드리지도 않는다.
대신 "12345번 학생 정보 다 줘"는 학번·이름·학과·성적·등록일 묶음을 전부 뒤져 같은 위치 카드를 다섯 번 맞춰 꺼내야 한다.

이 비유의 핵심은 **"무엇을 한 덩어리로 묶어 두느냐"** 다.
행기반은 한 행을, 컬럼기반은 한 컬럼을 한 덩어리로 본다.
그래서 점조회(OLTP)와 대량 집계(OLAP)의 유불리가 정확히 뒤집힌다.

---

## 4. 메커니즘: 컬럼 저장이 OLAP에서 빠른 두 가지 이유

### (1) I/O 절감 — 필요한 컬럼만 읽는다

컬럼 저장이 분석에서 빠른 1차 이유는 **불필요한 I/O를 안 한다**는 것이다.
쿼리에 `country`와 `revenue` 두 컬럼만 필요하면, 그 두 묶음만 디스크에서 읽는다.
나머지 수십 개 컬럼은 디스크에서 끌어올 일이 아예 없다.

행기반은 그럴 수가 없다.
디스크 I/O가 고정 크기 블록 단위라, 행 전체가 한 블록에 묶여 있으니 두 컬럼만 쓰는 쿼리에도 나머지 컬럼을 메모리로 함께 끌어와야 한다.
컬럼 수가 많을수록 이 낭비가 커진다.

ClickHouse 공식 문서는 이 효과를 1억 행 처리 쿼리를 92ms에 끝내는 것으로 보여준다 — 초당 약 10억 행 이상, 전송 처리량으로 약 7GB/s 수준이다 (공식 문서 기준).

### (2) 압축 — 같은 타입이 모여 있으니 잘 줄어든다

2차 이유는 압축이다.
같은 컬럼의 값은 타입이 같고 값도 서로 비슷하다 — 날짜는 날짜끼리, 국가코드는 국가코드끼리.
**비슷한 값이 인접하면 압축률이 올라가고, 압축이 잘 되면 디스크에서 읽을 바이트가 줄어 I/O가 또 줄고, 결국 쿼리도 빨라진다.**

ClickHouse는 이 점을 끝까지 활용한다.
범용 코덱(LZ4 기본, LZ4HC, ZSTD)에 더해, 데이터 특성에 맞춘 특화 코덱을 **컬럼 단위로** 지정할 수 있다.
정수 시계열엔 Delta·DoubleDelta(차분), 부동소수점엔 Gorilla·FPC·ALP, 정수엔 GCD·T64 같은 식이다 (공식 문서 기준).
게다가 `CODEC(Delta, ZSTD)`처럼 코덱을 파이프라인으로 결합할 수도 있다 — 먼저 델타로 줄이고 그 결과를 다시 ZSTD로 압축한다.
이건 행기반 RDB에는 사실상 없는 자유도다.

### (3) 벡터화 실행 — 값 하나가 아니라 컬럼 배열 통째로

세 번째는 처리 방식이다.
ClickHouse는 데이터를 **Block**(여러 컬럼 청크의 묶음) 단위로 처리하는 **벡터화 실행(vectorized execution)** 을 한다 (공식 문서 기준).
값을 한 개씩 처리하는 대신 컬럼 배열 전체에 한 번에 연산을 적용한다.
이렇게 하면 CPU 캐시를 잘 쓰고 SIMD(한 명령으로 여러 데이터 처리)를 활용할 수 있다.

내부적으로 Block은 `(컬럼, 데이터타입, 컬럼명)` 트리플의 집합이고, `WHERE`는 컬럼 필터로, `ORDER BY`는 컬럼 재배열(permute)로 처리된다 — 연산이 행 단위가 아니라 컬럼 배치 단위로 돈다.

### MergeTree와 sparse index — OLTP의 B-tree와 대조되는 지점

이 모든 게 ClickHouse의 주 저장 엔진 MergeTree 위에서 돌아간다(3·4장).
데이터는 기본키로 정렬된 'part'에 쌓이고, 삽입할 때마다 새 part가 생긴 뒤 백그라운드 스레드가 주기적으로 **병합(merge)** 한다 — 이름의 유래다.

여기서 기본키는 OLTP의 B-tree처럼 **모든 행을 인덱싱하지 않는다.**
대신 granule(기본 8192행) 범위를 가리키는 **sparse index**다 (공식 문서 기준, 버전에 따라 조정 가능).
`primary.idx`(N번째 행마다 기본키 값)와 마크 파일로 불필요한 범위를 통째로 건너뛴다(data skipping).
"모든 행을 정밀 인덱싱"하는 OLTP 모델과 "범위만 거칠게 잡고 건너뛰는" OLAP 모델의 대조가 바로 이 지점이다.

---

## 5. 코드로 보는 차이

### 행기반 OLTP — 한 행을 통째로 빠르게

```sql
-- 행기반 RDB: PK로 한 사용자 레코드 전체를 점조회/갱신 (OLTP 강점)
SELECT id, name, email, balance, updated_at
FROM users
WHERE id = 12345;            -- 한 행의 모든 컬럼이 디스크에 연속 → 1회 I/O로 전체 행

UPDATE users SET balance = balance - 100 WHERE id = 12345;  -- 트랜잭션 점수정 유리
```

한 행이 디스크에 연속으로 있으니, PK로 찾은 한 레코드를 한 번에 읽고 고치는 게 자연스럽다.

### 컬럼기반 OLAP — 필요한 컬럼만 골라 대량 집계

```sql
-- ClickHouse: 1억+ 행에서 country, revenue 두 컬럼만 디스크에서 읽음
-- (다른 수십 개 컬럼은 I/O 없음 → 컬럼 저장의 핵심 이점)
SELECT country,
       sum(revenue)      AS total,
       avg(revenue)      AS avg_rev,
       count()           AS events
FROM events
WHERE event_date >= '2026-01-01'
GROUP BY country
ORDER BY total DESC
LIMIT 10;   -- 벡터화 실행 + sparse 기본키로 event_date 범위 밖 granule 스킵
```

같은 SQL 문법이지만, 디스크에서 실제로 읽는 건 `country`·`revenue`·`event_date` 정도뿐이다.

### 컬럼별 압축 코덱 — 행기반엔 없는 자유도

```sql
-- 컬럼마다 데이터 특성에 맞는 압축 코덱 지정 (행기반 RDB엔 없는 컬럼 저장의 자유도)
CREATE TABLE metrics
(
    ts        DateTime  CODEC(DoubleDelta, ZSTD),   -- 일정 간격 타임스탬프 → 델타의 델타
    sensor_id UInt32    CODEC(T64, LZ4),            -- 작은 정수 전치 압축
    value     Float64   CODEC(Gorilla)              -- 부동소수점 XOR 압축
)
ENGINE = MergeTree
ORDER BY (sensor_id, ts);    -- 기본키 = sparse index (granule 8192행 단위)
```

컬럼마다 그 컬럼에 맞는 압축을 따로 거는 것 — 컬럼을 한 덩어리로 보는 저장 방식이라야 가능한 일이다.

---

## 6. 한눈에 보는 비교표

| 구분 | 행기반 (Row-oriented) | 컬럼기반 (Column-oriented) |
|---|---|---|
| 디스크 저장 단위 | 한 행의 모든 컬럼을 연속 저장 | 같은 컬럼 값들을 한데 모아 저장 |
| 주 워크로드 | OLTP (트랜잭션) | OLAP (분석·집계) |
| 강한 작업 | 점조회, 신규 행 삽입, 한 행 수정 | 특정 컬럼 프로젝션, 대량 집계 |
| 약한 작업 | 대량 컬럼 집계(불필요 컬럼까지 읽음) | 단일 행 전체 점조회(여러 묶음 조합) |
| 압축 | 상대적으로 불리(이종 데이터 혼재) | 유리(동질 데이터 인접 → 크기 작음) |
| SIMD/벡터화 | 제한적 | 컬럼 배열 단위로 유리 |
| 대표 시스템 | PostgreSQL, MySQL, Oracle, CSV, Avro | ClickHouse, BigQuery, Redshift, Snowflake, Parquet, ORC, DuckDB |
| 인덱스 성향 | 모든 행 정밀 인덱싱(B-tree 등) | sparse index + data skipping |
| 이론·기원 | 관계형 모델(Codd, 1970) | DSM(1985)·C-Store(2005)→Vertica |

이 표의 출처는 ClickHouse 공식 문서와 컬럼 지향 DB 일반 정리다.
행기반/컬럼기반 대표 시스템 목록도 같은 정리 기준이다.

---

## 7. "왜 이렇게?" — 트레이드오프를 한 문장으로

저장 방식 하나가 이렇게 멀리 가는 이유는, **"무엇을 한 덩어리로 묶느냐"가 디스크 I/O 패턴을 결정하고, I/O 패턴이 그 DB가 잘하는 일을 결정**하기 때문이다.

행을 묶으면 한 레코드를 통째로 다루는 게 싸지고, 컬럼을 가로지르는 집계가 비싸진다.
컬럼을 묶으면 그 반대다.
어느 쪽도 "더 우월한" 게 아니라, **풀려는 문제가 다르다.**

그래서 결제·주문·재고 같은 트랜잭션 시스템엔 여전히 행기반 RDB가 들어가고(1장에서 본 "ClickHouse를 언제 안 쓰나"의 이유와 맞닿는다 — 단일행 point lookup, mutation 기반 비동기 UPDATE/DELETE, multi-statement ACID 트랜잭션 부재), 로그·이벤트·메트릭 분석엔 ClickHouse 같은 컬럼기반 OLAP DB가 들어간다.

여기서 한 가지 솔직하게 덧붙이면, 최근에는 한 시스템이 행과 컬럼을 함께 다루려는 하이브리드(HTAP) 흐름도 있다.
다만 그 구체적인 시스템과 트레이드오프는 이번 정리 범위에서 충분히 검증하지 못해서, 단정하지 않고 "그런 방향도 있다" 정도로만 적어둔다.

---

## 8. 언제 무엇을 — 선택 가이드

- **행기반(OLTP RDB)을 쓸 때**
  - 한 레코드를 PK로 찾아 통째로 읽고 쓴다 (사용자 프로필, 주문 한 건).
  - 잦은 단일행 삽입·수정·삭제, 강한 트랜잭션 일관성(여러 문장을 한 트랜잭션으로).
  - 데이터 규모가 "수십억 행 스캔"이 아니라 "정확한 몇 행을 빠르게".

- **컬럼기반(ClickHouse 같은 OLAP DB)을 쓸 때**
  - 수백만~수십억 행을 가로질러 합·평균·카운트로 집계한다 (대시보드, 리포트).
  - 전체 컬럼 중 일부만 쓰는 쿼리가 많다.
  - 원본(비집계) 데이터를 그대로 쌓되, 임의 차원으로 빠르게 분석하고 싶다.
  - 시계열·로그·클릭스트림처럼 컬럼별 압축이 크게 먹히는 데이터.

요약하면 — **행을 통째로 만지면 행기반, 컬럼을 가로질러 집계하면 컬럼기반**이다.

---

다음 18장에서는 이 컬럼기반 OLAP 안에서도 ClickHouse가 어디쯤 서 있는지 — 실시간 분석 DB 지형(Druid·Pinot·Snowflake·BigQuery 등)과의 비교 — 를 정리한다.
1장에서 본 "ClickHouse를 언제 안 쓰나"가 이 17장의 트레이드오프와 18장의 지형도 사이 어딘가에서 그 자리를 찾는 셈이다.

---

## 참고

- ClickHouse 공식 문서 — Intro (행기반 저장 설명, 1억 행 92ms): <https://clickhouse.com/docs/intro>
- ClickHouse 공식 문서 — Why ClickHouse is so fast (압축 코덱·벡터화 Block·MergeTree sparse index): <https://clickhouse.com/docs/concepts/why-clickhouse-is-so-fast>
- ClickHouse 공식 문서 — History (Yandex.Metrica 기원·비집계 설계·120억 이벤트): <https://clickhouse.com/docs/about-us/history>
- ClickHouse — What is OLAP: <https://clickhouse.com/resources/engineering/what-is-olap>
- Wikipedia — Online Analytical Processing (Codd 1970/1993): <https://en.wikipedia.org/wiki/Online_analytical_processing>
- Wikipedia — Column-oriented DBMS (행/컬럼 트레이드오프·대표 시스템): <https://en.wikipedia.org/wiki/Column-oriented_DBMS>
- Wikipedia — C-Store (DSM 1985·C-Store 2005·Vertica): <https://en.wikipedia.org/wiki/C-Store>
- Wikipedia — SAP IQ (Sybase IQ 기원): <https://en.wikipedia.org/wiki/SAP_IQ>
- Wikipedia — ClickHouse (오픈소스화·법인화·채택 기업): <https://en.wikipedia.org/wiki/ClickHouse>
- C-Store 논문 (Abadi et al., VLDB): <https://vldb.org/2015/wp-content/uploads/2015/09/abadi.pdf>
