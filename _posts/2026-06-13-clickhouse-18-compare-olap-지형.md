---
title: "ClickHouse 완전 정복 18장 — 비교: OLAP·실시간 분석 DB 지형, ClickHouse는 어디에 서 있나"
date: 2026-06-13 23:41:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "비교"]
---
17장에서 우리는 데이터베이스 세계가 행기반(OLTP)과 컬럼기반(OLAP)으로 갈라진 뿌리를 봤다.
ClickHouse는 그 컬럼기반 갈래에 속한다.
그런데 컬럼기반 OLAP 갈래 안에도 식구가 많다.
Druid, Pinot, Doris, StarRocks, DuckDB, BigQuery, Snowflake, Elasticsearch — 이름만 들어도 머리가 복잡해진다.

이번 장은 한 걸음 더 물러나 **그 지형도 전체**를 그린다.
이들은 다 "분석 빠르게 한다"고 말하지만, 자란 환경도 다르고 잘하는 일도 다르다.
ClickHouse가 어디쯤 서 있는지 알려면, 옆 동네들이 무엇을 풀려고 태어났는지부터 봐야 한다.

미리 한 가지 못박아 둔다.
이 장의 "누가 더 빠르다"류 수치는 대부분 **각 제품을 만든 회사가 낸 벤치마크**라서 자기에게 유리하게 측정됐을 수밖에 없다.
그래서 여기서는 "ClickHouse가 X배 빠르다"처럼 단정하지 않고, **"이 시스템은 어떤 문제에 1순위로 어울리는가"** 라는 카테고리 포지셔닝으로만 정리한다.

---

## 30초 요약

- OLAP 분석 DB는 한 덩어리가 아니라 **풀려는 문제별로 갈라진 여러 카테고리**다.
- **ClickHouse**: 단일 대형 테이블 집계·로그/범용 OLAP에 강한 범용 실시간 분석 DB.
- **Druid / Pinot**: 실시간 수집 + 유저 대면(고동시성·저지연) 분석에 특화된 시계열 계열.
- **Doris / StarRocks**: 조인 무거운 스타 스키마·lakehouse 쿼리에 강한 MPP 계열.
- **DuckDB**: 서버 없이 프로세스 안에서 도는 임베디드 분석 엔진("분석용 SQLite").
- **BigQuery / Snowflake**: 대규모 정적 스캔·데이터 웨어하우징·BI 리포팅용 서버리스/클라우드 웨어하우스.
- **Elasticsearch**: 로그·전문 검색 엔진.
  집계도 되지만 대규모 집계는 컬럼형 OLAP보다 불리.
- 결론은 17장과 같다 — **"뭐가 제일 빠르냐"가 아니라 "내 워크로드가 어느 카테고리냐"** 가 진짜 질문이다.

---

## 1. 먼저 지도의 축부터: 분석 DB를 가르는 네 가지 용어

지형을 보기 전에, 자주 나오는 단어 넷을 정리하고 가자.

- **OLAP** (Online Analytical Processing): 수백만~수십억 행을 스캔해 합·평균·카운트로 **집계**하는 일.
  17장에서 봤듯 컬럼 기반 저장이 여기에 유리하다 — 필요한 컬럼만 읽고 압축이 잘 먹힌다.
- **OLTP** (Online Transaction Processing): 소수 행을 밀리초 단위로 **삽입·갱신**하는 일.
  행 기반이 유리.
- **real-time analytics DB** (실시간 분석 DB): 방금 들어온 **신선한(fresh)** 데이터에 대해 **1초 미만(sub-second)** 으로, 그리고 **많은 사용자가 동시에** 쿼리를 던질 수 있는 분석 DB.
  Druid·Pinot이 대표적으로 이 자리를 노리고 태어났다.
- **HTAP** (Hybrid Transactional/Analytical Processing): 한 시스템에서 OLTP와 OLAP을 **둘 다** 하려는 시도.
  행 저장과 컬럼 저장을 한데 결합한다.
- **lakehouse**: 데이터 레이크(S3 같은 값싼 객체 저장소) 위에 웨어하우스급 **테이블 관리**(트랜잭션·스키마·버저닝)를 얹은 구조.
  Iceberg·Delta·Hudi 같은 오픈 테이블 포맷이 이를 떠받친다.

HTAP·lakehouse 정의는 공식 1차 문서보다 2차 정리에 의존한 부분이 있어, 여기서는 큰 그림만 잡는 용도로 쓴다.

이 축들을 머리에 넣고 보면, 아래 시스템들이 "왜 서로 다른가"가 보인다.

---

## 2. 비유: 같은 '빠른 차'라도 용도가 다르다

이건 어디까지나 이해를 돕는 비유다 — 실제 차이는 저장 구조·인덱스·배포 모델이다.

분석 DB들을 "빠른 자동차"라고 해보자.
다 빠르지만 빠른 방향이 다르다.

- **ClickHouse**는 짐도 많이 싣고 장거리도 잘 가는 **만능 고성능 세단**에 가깝다.
  로그 한 트럭분을 통째로 집계하는 데 강하다.
- **Druid·Pinot**은 신호등(실시간 유입) 많은 도심에서 **수백 대가 동시에 출발해도 막히지 않는** 도심형이다.
  유저 대면 대시보드가 여기다.
- **Doris·StarRocks**는 여러 화물(여러 테이블)을 **이어 붙여 끌고 가는(조인)** 데 특화된 트레일러다.
- **DuckDB**는 아예 차고에서 안 나오는 **집 안의 미니 분석기** — 서버를 따로 띄울 필요가 없다.
- **BigQuery·Snowflake**는 **거대한 물류 창고** — 어마어마한 정적 화물을 스캔·정리하는 데 강하지만 실시간 즉답에는 덜 어울린다.
- **Elasticsearch**는 애초에 분석차가 아니라 **검색차** — 책 더미에서 단어를 찾는 데 특화됐다.

핵심은 "빠르냐"가 아니라 **"어떤 빠름이냐"** 다.
이제 각 차의 출생 배경으로 들어가 보자.

---

## 3. 메커니즘: 각 시스템은 어떤 문제에서 태어났나

### ClickHouse — 범용 실시간 OLAP의 기준점

17장에서 봤듯 ClickHouse는 2008~2009년 Yandex의 웹 분석 플랫폼 Metrica를 위해 시작돼, 2012년 프로덕션, 2016년 Apache 2.0으로 오픈소스화됐다.
MergeTree(3장)를 중심으로 **단일 대형 테이블을 대량 집계**하는 데 강하다.
로그·이벤트·메트릭 같은 범용 OLAP에서 1순위로 꼽히는 자리다.

전통적으로 약점으로 지적된 건 **복잡한 조인**과 **초고동시성 유저 대면** 쪽이었는데, 최근 버전(공개적으로 확인되는 범위에선 25.09/25.10대)에서 개선이 이뤄진 것으로 알려져 있다.
ClickHouse는 자사 자료에서 Postgres 대비 압축이 10~20배라고 주장하는데, 이 배수는 자사 출처라 그대로 단정하기보다 "압축이 크게 먹힌다" 정도로 받아들이는 게 안전하다.

### Apache Druid — 시계열 탐색과 실시간 수집

Druid는 2011년 Metamarkets에서 시작돼 2012년 오픈소스, 2018년 Apache 재단으로 갔다.
데이터를 **시간으로 파티셔닝한 segment**에 쌓고 **비트맵 인덱스**를 얹어, 시계열 데이터를 빠르게 탐색하고 실시간으로 수집하는 데 강하다.
"방금 들어온 데이터를 시간 축으로 깎아 본다"는 작업에서 1순위로 꼽힌다.

대신 전통적으로 행 단위 UPDATE/DELETE가 없어, 데이터를 고치려면 해당 구간을 **배치로 다시 수집**해야 한다.
또 다양한 노드 역할과 ZooKeeper 의존으로 클러스터 운영이 복잡한 편이라는 평이 있다 — 다만 이 "운영 복잡" 서술엔 경쟁사 관점이 섞여 있을 수 있으니 절대적 단점으로 못박지는 않는다.

### Apache Pinot — 극도의 동시성, 유저 대면

Pinot은 2013년 LinkedIn에서 만들어져 2015년 오픈소스, 2021년 Apache TLP가 됐고, 2019년 그 위에 StarTree라는 회사가 창업됐다.
설계 목표가 또렷하다 — **수많은 최종 사용자가 동시에 던지는 저지연 쿼리**.
LinkedIn의 "누가 내 프로필을 봤나" 같은 유저 대면 분석이 전형적인 무대다.
**Star-Tree 인덱스**로 미리 부분 집계를 만들어 동시성·지연을 잡고, 실시간 upsert도 지원한다.

upsert 우위 같은 주장은 Pinot 벤더(StarTree) 출처라, "유저 대면 고동시성에 강하다"는 카테고리 포지셔닝까지만 받아들인다.
Pinot 역시 여러 컴포넌트로 이뤄진 클러스터라 구성 요소가 많다.

### Apache Doris / StarRocks — 조인 무거운 MPP

Doris는 Baidu의 Palo로 시작해 2017년 오픈소스, 2022년 6월 Apache TLP가 됐다.
StarRocks는 2020년 Doris를 포크해 코드의 약 80%를 새로 쓴 갈래로, 2022년 Linux 재단으로 갔다.

이 계열의 강점은 **비용 기반 옵티마이저(CBO) + 벡터화**로 **조인이 무거운 스타 스키마**와 **lakehouse(Iceberg 등) 쿼리**를 잘 처리한다는 데 있다.
네이티브 upsert도 지원한다.
StarRocks 자사 자료엔 ClickHouse 대비 2.3~10배 빠르다는 배수가 나오는데, 이건 자사 측정이고 공개 표준 벤치마크(ClickBench)에 제출된 정량 순위가 없어 **그대로 인용하지 않는다** — "조인·스타 스키마 워크로드에 강한 카테고리"로만 본다.

### DuckDB — 서버 없는 임베디드 분석 엔진

DuckDB는 네덜란드 CWI의 Mark Raasveldt와 Hannes Mühleisen이 만들어 2019년 처음 릴리스됐다.
스스로를 **"분석용 SQLite(SQLite for Analytics)"** 라 부른다.
즉 **인프로세스(임베디드)** 로 돈다 — 별도 서버를 띄우는 게 아니라, 내 프로그램(파이썬·R 등) **안에서** 컬럼형 벡터화 실행을 한다.
클라이언트-서버 간 데이터 전송이 없어, 단일 노드에서 로컬 데이터를 분석하는 데 최적이다.
공개 표준 벤치마크(ClickBench)에서도 ClickHouse 다음 2위권의 단일 노드 성능을 보인다.

ClickHouse가 "서버 클러스터"라면 DuckDB는 "내 노트북 프로세스 안"이라는 점에서, 둘은 경쟁이라기보다 **무대가 다르다.**

### BigQuery / Snowflake — 서버리스·클라우드 웨어하우스

BigQuery는 2010년 출시된 Google의 완전 **서버리스** 웨어하우스다.
Dremel 기술을 기반으로, 사용자는 슬롯(slot) 풀에 SQL만 제출하고 스토리지와 컴퓨트는 분리돼 있다 — 서버를 직접 굴린다는 감각이 없다.
Snowflake는 스토리지·컴퓨트·서비스의 **3계층 분리** 구조로, "가상 웨어하우스"라는 컴퓨트 단위를 사용자가 직접 골라 구성한다.

둘 다 **대규모 정적 데이터 스캔·데이터 웨어하우징·BI 리포팅**에 1순위로 어울린다.
반면 신선한 데이터에 1초 미만으로 응답해야 하는 **실시간 유저 대면 서빙**에는 덜 어울린다는 평이 있는데, 이 "서빙 부적합" 서술은 ClickHouse 자사 출처라 절대적 한계로 단정하지 않고 "강점 무대가 다르다" 정도로 본다.

### Elasticsearch — 검색 엔진이 분석을 겸할 때

Elasticsearch는 Lucene **역색인(inverted index)** 기반 검색 엔진이다.
로그 검색과 전문(full-text) 검색에서 1순위다.
집계를 위한 doc values 구조도 있지만, 인덱싱이 느리고 원본 대비 스토리지가 더 드는 편이라 **대규모 집계에서는 컬럼형 OLAP보다 비효율적**이라는 평이 있다.
다만 "원본 대비 2~3배 스토리지" 같은 수치는 2차·경쟁 출처라 단정하지 않고, **"검색엔 강하지만 대규모 집계엔 컬럼형 OLAP이 더 맞다"** 는 방향만 적어둔다.

---

## 4. ClickBench라는 공용 줄자 — 그리고 그 한계

이쯤에서 "그래서 객관적으로 누가 빠른가"가 궁금할 텐데, 가장 널리 인용되는 공용 벤치마크가 **ClickBench**다.
이름에서 짐작되듯 **ClickHouse가 만든** 벤치마크라는 점을 먼저 못박는다.

ClickBench의 구성은 이렇다 (공개된 벤치마크 정의 기준).

- 익명화된 웹 분석 데이터 **99,997,497행**(약 1억 행), **43개 쿼리**.
- cold/hot run(차가운/뜨거운 캐시) + 적재(load) 시간 + 스토리지 크기를 함께 측정.
- 60개 이상의 DBMS를 같은 표준 하드웨어(AWS `c6a.4xlarge`, 500GB)에서 비교.
- **벤더가 자발적으로 제출**하는 방식 — 즉 목록에 없으면 "졌다"가 아니라 "제출하지 않았다"는 뜻이다.

2026년 시점의 한 스냅샷 예시는 다음과 같다 (ClickBench 공개 수치 기준 · 측정 주체가 ClickHouse라 편향 가능).

| 시스템 | 대략적 처리 시간 | 비고 |
|---|---|---|
| ClickHouse | 약 148ms | 43쿼리 중 약 32개에서 최고 성능 |
| DuckDB | 약 348ms | 단일 노드 2위권 |
| Druid | 약 1.77s | 일부 쿼리(약 14개) 실패 |
| Pinot | 약 3.02s | 일부 쿼리(약 5개) 실패 |
| StarRocks / Doris | 미제출 | 정량 순위 없음 |

이 표는 "절대 진실"이 아니라 **대략의 tier(급) 차이**를 보는 용도다.
측정 주체가 ClickHouse이고, 벤더 자발 제출이라 표에 빠진 강자가 있을 수 있으며, 단일 노드 표준 HW라 분산·고동시성 같은 다른 축은 담지 못한다.
그래서 이 숫자로 "X가 Y보다 무조건 빠르다"고 결론 내리는 건 위험하다 — 카테고리가 다른 시스템을 한 줄에 세운 줄자라는 점을 늘 같이 기억해야 한다.

---

## 5. 한눈에 보는 지형 비교표

아래 표는 위에서 본 출생 배경과 강점을 카테고리로 정리한 것이다 — "성능 순위"가 아니라 **"어느 문제에 1순위로 어울리는가"** 를 본다.

| 시스템 | 카테고리 | 출생 배경 | 1순위로 어울리는 일 | 상대적 약점/유의점 |
|---|---|---|---|---|
| ClickHouse | 범용 실시간 OLAP | Yandex Metrica (2009~) | 단일 대형 테이블 집계, 로그/이벤트/메트릭 범용 OLAP | 복잡 조인·초고동시성은 전통적 약점(최근 개선) |
| Apache Druid | 실시간 시계열 분석 | Metamarkets (2011) | 시간 축 탐색, 실시간 수집 대시보드 | UPDATE/DELETE 부재(배치 재수집), 운영 컴포넌트 다수 |
| Apache Pinot | 유저 대면 실시간 분석 | LinkedIn (2013) | 극도 동시성·저지연 유저 대면 쿼리 | 다컴포넌트 클러스터, 벤더 출처 주장 다수 |
| Doris / StarRocks | MPP 분석 DB | Baidu Palo / Doris 포크 | 조인 무거운 스타 스키마, lakehouse(Iceberg) 쿼리 | ClickBench 미제출(정량 순위 미확인) |
| DuckDB | 임베디드 분석 | CWI (2019) | 단일 노드·인프로세스 로컬 분석 | 분산·대규모 동시 서빙 무대 아님 |
| BigQuery / Snowflake | 클라우드 웨어하우스 | Google(2010) / Snowflake | 대규모 정적 스캔, 웨어하우징, BI | 실시간 유저 대면 서빙엔 덜 어울림(자사 비교 출처) |
| Elasticsearch | 검색 엔진 | Lucene 기반 | 로그·전문 검색 | 대규모 집계는 컬럼형 OLAP 대비 비효율 |

이 표의 근거는 각 프로젝트 위키·공식 문서, 그리고 일부는 벤더 블로그다.
벤더 블로그 출처가 섞인 항목(특히 "X가 더 낫다"류)은 위 본문에서 톤을 낮춰 적었다.

---

## 6. "왜 이렇게?" — 한 카테고리로 다 못 덮는 이유

지형이 이렇게 갈라진 이유는 17장의 결론을 한 층 더 민 것이다.

저장을 행으로 묶느냐 컬럼으로 묶느냐가 OLTP/OLAP을 갈랐다면(17장),
**같은 컬럼형 OLAP 안에서도 "무엇을 추가로 최적화하느냐"** 가 카테고리를 또 가른다.

- "신선한 데이터를 시간 축으로, 많은 사람이 동시에" → Druid·Pinot이 segment·Star-Tree 인덱스로 그쪽을 깎았다.
- "여러 테이블을 이어 붙이는 조인을 무겁게" → Doris·StarRocks가 CBO·MPP로 그쪽을 깎았다.
- "서버조차 띄우기 싫다, 내 프로세스 안에서" → DuckDB가 임베디드로 그쪽을 깎았다.
- "어마어마한 정적 데이터를 클라우드에서 손 안 대고" → BigQuery·Snowflake가 서버리스/3계층으로 그쪽을 깎았다.

각자가 어느 한 축을 깎으면 다른 축이 비싸진다 — 17장에서 본 트레이드오프의 법칙이 여기서도 그대로 작동한다.
그래서 "하나가 모두를 이긴다"는 일이 일어나지 않는다.
ClickHouse는 이 지형에서 **특정 축에 올인하지 않은, 범용 실시간 OLAP의 기준점** 쯤에 서 있다고 보면 이해가 쉽다.

---

## 7. 언제 무엇을 — 선택 가이드

- **ClickHouse**: 로그·이벤트·메트릭처럼 **단일 대형 테이블을 임의 차원으로 대량 집계**.
  범용 실시간 OLAP의 기본값.
- **Druid**: **시간 축 탐색 + 실시간 수집 대시보드**.
  방금 들어온 시계열을 빠르게 깎아 본다.
- **Pinot**: **유저 대면(외부 고객) 분석**으로 **수많은 동시 사용자 + 저지연**이 핵심일 때.
- **Doris / StarRocks**: **조인이 무거운 스타 스키마**, 또는 **lakehouse(Iceberg) 위 쿼리**가 중심일 때.
- **DuckDB**: **단일 노드에서, 서버 없이** 파이썬/R 안에서 로컬 분석할 때.
  클러스터가 필요 없을 때.
- **BigQuery / Snowflake**: **대규모 정적 데이터 웨어하우징·BI 리포팅**, 인프라 운영을 클라우드에 맡기고 싶을 때.
- **Elasticsearch**: **전문 검색·로그 검색**이 주목적일 때.
  단, 대규모 집계가 중심이면 컬럼형 OLAP을 본다.

요약하면 — **"제일 빠른 하나"를 고르는 게 아니라, 내 워크로드가 어느 카테고리에 떨어지는지를 먼저 정하는 것**이 순서다.

---

이로써 비교 두 장(17~18)을 마친다.
17장에서 행/컬럼이라는 큰 갈래를 봤고, 18장에서 컬럼형 OLAP 안의 지형을 봤다.
1장에서 본 "ClickHouse를 언제 안 쓰나"가 이 둘 사이 어딘가 — 행기반이 맞는 트랜잭션 영역, 그리고 컬럼형 안에서도 다른 축이 더 맞는 영역 — 에서 그 자리를 찾는다.
시리즈 전체를 한 번 더 점검하고 싶다면 15장 마스터 체크로 돌아가면 된다.

---

## 참고

- ClickHouse — How to choose a database for real-time analytics (카테고리 포지셔닝): <https://clickhouse.com/resources/engineering/how-to-choose-a-database-for-real-time-analytics-in-2026>
- ClickHouse — Fastest OLAP databases / ClickBench 해설: <https://clickhouse.com/resources/engineering/fastest-olap-databases>
- ClickBench (벤치마크 정의·구성·결과): <https://github.com/ClickHouse/ClickBench>
- Wikipedia — ClickHouse: <https://en.wikipedia.org/wiki/ClickHouse>
- Wikipedia — Apache Druid: <https://en.wikipedia.org/wiki/Apache_Druid>
- Apache Druid — Segments 설계 문서: <https://druid.apache.org/docs/latest/design/segments/>
- Wikipedia — Apache Pinot: <https://en.wikipedia.org/wiki/Apache_Pinot>
- StarTree — The story of Apache Pinot (LinkedIn 기원): <https://startree.ai/resources/launching-at-linkedin-the-story-of-apache-pinot/>
- Apache Doris — Announcing TLP: <https://doris.apache.org/blog/Annoucing/>
- DB of Databases — StarRocks: <https://dbdb.io/db/starrocks>
- DuckDB — SIGMOD 2019 demo paper: <https://duckdb.org/pdf/SIGMOD2019-demo-duckdb.pdf>
- MotherDuck — DuckDB Book Ch.1 summary: <https://motherduck.com/duckdb-book-summary-chapter1/>
- Airbyte — Snowflake vs BigQuery: <https://airbyte.com/data-engineering-resources/snowflake-vs-bigquery>
- Airbyte — ClickHouse vs Elasticsearch: <https://airbyte.com/data-engineering-resources/clickhouse-vs-elasticsearch>
- CelerData — HTAP glossary: <https://celerdata.com/glossary/hybrid-transactional-analytical-processing>
