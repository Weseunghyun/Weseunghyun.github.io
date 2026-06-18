---
title: "ClickHouse 완전 정복 1장 — ClickHouse란 무엇이고, 언제 쓰면 안 되나"
date: 2026-06-13 23:58:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "입문"]
---
## 한 줄 정의부터

공식 문서는 ClickHouse를 이렇게 정의한다.

> **"a high-performance, column-oriented SQL database management system (DBMS) for online analytical processing (OLAP)."**
> (고성능, 컬럼 지향 SQL 데이터베이스 관리 시스템, OLAP용.)

이 한 문장에 ClickHouse의 정체가 다 들어 있다.
네 조각으로 뜯어보자.

| 조각 | 뜻 | 어디서 다루나 |
|---|---|---|
| high-performance | 초당 수억~수십억 행 스캔 | 2장 (왜 빠른가) |
| column-oriented | 데이터를 행이 아니라 컬럼 단위로 저장 | 2장 |
| SQL DBMS | 익숙한 SQL로 질의 | 시리즈 전반 |
| for OLAP | 분석(대량 집계) 전용, 거래용 아님 | 이 장 |

앞 셋(고성능·컬럼·SQL)은 "어떻게"에 관한 것이고,
마지막 **for OLAP**이 "무엇을 위한 것인가" — 이 도구의 **용도와 한계**를 못박는다.
그래서 이 장은 마지막 조각, **"분석용이라는 게 실제로 무슨 뜻이고, 그래서 뭘 못하는가"**에 집중한다.

## ClickHouse가 잘하는 일

ClickHouse의 강점은 한 문장으로 **"넓게 훑어 요약하기"**다.
구체적으로는 이런 모양의 작업이다(공식 문서가 드는 전형적 OLAP 패턴).

- **많은 행을 스캔** — 수백만~수십억 행.
- **적은 컬럼만 사용** — 전체 컬럼 중 집계에 필요한 몇 개만.
- **집계·필터·그룹화** — `SUM`, `COUNT`, `GROUP BY`, 시간대별 추이.
- **읽기 위주** — 한 번 쌓고 여러 번 읽는다.
  자주 고치지 않는다.

전형적 사용처는 이렇다.

| 영역 | 예 |
|---|---|
| 로그·관측(observability) | 애플리케이션 로그, 메트릭, 트레이스 대량 저장·조회 |
| 실시간 분석 대시보드 | 사용자 행동, 트래픽, 매출 추이 |
| 이벤트/클릭스트림 | 웹·앱 이벤트 수집 후 집계 |
| 시계열 | IoT 센서, 모니터링 시계열 |

공식 벤치마크가 드는 장면이 이 강점을 압축한다 — **1억 행을 92밀리초에**, 초당 10억 행을 넘는 처리량으로 훑는다(공식 문서 기준).
이게 가능한 이유가 2장의 주제다.

## ClickHouse를 쓰면 안 되는 일 — OLTP

여기가 이 장의 핵심이다.
ClickHouse는 빠르지만 **만능이 아니다.**
오히려 **빠른 이유와 못하는 이유가 같은 뿌리**에서 나온다.
"넓게 훑기"에 모든 설계를 몰아준 대가로, **거래용(OLTP) 작업은 구조적으로 약하다.**

공식 문서와 여러 1차 자료가 일관되게 가리키는, ClickHouse가 약한 지점들이다.

### 1) 단일 행 콕 집어 읽기(point lookup)가 느리다

ClickHouse의 인덱스는 **sparse(성긴) 인덱스**다(4장에서 자세히).
모든 행을 가리키는 게 아니라 **8192행 묶음(granule)마다 하나씩**만 가리킨다.
그래서 "id = 12345인 그 한 줄"을 찾을 때도,
정확히 그 줄로 점프하지 못하고 **그 줄이 든 8192행 묶음 전체를 읽어** 그 안에서 찾는다.
MySQL·PostgreSQL의 B-tree 인덱스가 한 줄로 바로 점프하는 것과 다르다.
**sub-millisecond 단일행 조회는 ClickHouse의 목표가 아니다**(공식 문서가 OLTP와 대비해 명시).

### 2) UPDATE·DELETE가 즉시 반영되지 않는다

ClickHouse의 데이터 조각(part)은 **불변(immutable)**이다 — 한 번 쓰면 고치지 않는다(3장).
그래서 `UPDATE`/`DELETE`는 일반 DB처럼 그 자리에서 즉시 고치는 게 아니라,
**mutation**이라는 무거운 비동기 작업으로 처리된다.
요청을 받으면 백그라운드에서 **관련 part들을 통째로 다시 써** 반영하는 식이라, 빈번한 수정에 맞지 않는다.

> 비유: 일반 DB의 UPDATE가 장부의 한 줄을 지우개로 고치는 거라면, ClickHouse의 mutation은 그 줄이 든 **공책 한 권을 통째로 새로 베껴 쓰는 것**에 가깝다. 가끔 하는 대량 정정엔 쓸 수 있어도, 초당 수천 번 고치는 용도는 아니다.

### 3) 다중 문장 트랜잭션·강한 ACID가 없다

은행 이체 같은 작업은 "A 계좌에서 빼고 B 계좌에 더하는" 두 동작이 **전부 되거나 전부 안 되거나** 해야 한다(원자성).
일반 OLTP DB는 이걸 트랜잭션으로 보장한다.
ClickHouse는 **여러 문장을 묶는 multi-statement 트랜잭션과 그 수준의 ACID를 제공하지 않는다**(1차 자료들이 일관되게 지적).
행 단위 잠금(row-level lock)도 OLTP 방식으로 제공하지 않는다.

### 정리 — 한 표로

| 작업 | 적합한 도구 | ClickHouse |
|---|---|---|
| 주문 한 건 생성·수정·삭제 | OLTP (PostgreSQL 등) | ✋ 부적합 |
| "id = X" 단일행 즉시 조회 | OLTP | ✋ 느림 (granule 전체 읽음) |
| 계좌 이체 같은 트랜잭션 | OLTP | ✋ 없음 |
| 수십억 행 집계·추이 분석 | **OLAP (ClickHouse)** | ✅ 강점 |
| 로그·이벤트 대량 적재 후 조회 | **OLAP** | ✅ 강점 |

이게 **"ClickHouse를 OLTP DB 대체로 쓰면 안 된다"**는 말의 구체적 내용이다.
실무에서는 보통 **둘을 같이 쓴다** — 거래는 PostgreSQL 같은 OLTP가 처리하고,
거기서 나온 이벤트·로그를 ClickHouse로 흘려보내 분석한다.
한쪽으로 다른 쪽을 대체하려 하면 양쪽 다 불행해진다.

## 그래서 핵심은

ClickHouse를 제대로 쓰는 첫걸음은 역설적이게도 **"안 쓸 자리를 아는 것"**이다.

- ClickHouse는 **OLAP 전용** — 넓게 훑어 요약하는 데 특화됐다.
- 빠른 이유(컬럼 저장·sparse 인덱스·불변 part)가 **그대로** OLTP에 약한 이유다.
- 단일행 조회·빈번한 수정·트랜잭션이 필요하면 OLTP DB를 쓴다.
- 둘은 경쟁이 아니라 **역할 분담**이다.

다음 2장에서는, 그 "빠른 이유"의 정체 — **컬럼 저장·압축·벡터화·병렬화**가 어떻게 맞물려 1억 행 92ms를 만드는지 — 를 분해한다.

---

## 참고

- ClickHouse 공식 문서 — What is ClickHouse?
  (OLAP 정의, OLTP 대비): https://clickhouse.com/docs/intro
- ClickHouse Resource Hub — OLTP vs OLAP: https://clickhouse.com/resources/engineering/oltp-vs-olap
- ClickHouse 공식 문서 — Why is ClickHouse so fast?: https://clickhouse.com/docs/concepts/why-clickhouse-is-so-fast
