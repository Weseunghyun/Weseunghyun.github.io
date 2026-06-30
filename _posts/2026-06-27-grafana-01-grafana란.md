---
title: "Grafana 완전 정복 1장 — 저장 안 하는 시각화 레이어"
date: 2026-06-27 23:58:00 +0900
categories: ["Grafana", "Grafana 완전 정복"]
tags: ["grafana", "입문"]
---
## 1.1 한 줄 정체

Grafana는 데이터를 저장하지 않는 **시각화·알림 레이어**다.
여러 데이터소스를 붙여 한 화면(패널·대시보드)에 모으고, 알림을 평가·라우팅한다.
Grafana Labs가 만들었고, UI는 2014년 Kibana fork에서 출발했다("그래프 중심의 Kibana" → Grafana).
최신은 **Grafana 13**(GrafanaCON 2026)이다.

## 1.2 무엇을 저장하나

| 저장하는 것 | 저장 안 하는 것 |
|---|---|
| 대시보드·패널 정의 | 시계열(메트릭) |
| 유저·조직·팀 | 로그 |
| 데이터소스 설정·알림 규칙 | 트레이스 |

자체 DB(기본 SQLite, 프로덕션은 MySQL/Postgres 권장)에는 **메타데이터만** 있다.
실제 관측 데이터는 외부 데이터소스에 있고, Grafana는 쿼리만 위임한다(2장).

## 1.3 무엇을 붙일 수 있나

Grafana 13 기준 **170개 이상의 데이터소스**를 플러그인으로 붙인다.

- LGTM: Prometheus·Loki·Tempo (+ Mimir)
- 그 외: Elasticsearch·InfluxDB·SQL(MySQL/Postgres)·CloudWatch·Graphite…

이 중립성 — "저장은 각자, 보는 건 Grafana 하나" — 가 Grafana를 사실상 표준 대시보드 도구로 만들었다.

## 1.4 무엇을 하나

- **대시보드**: 패널(쿼리+시각화)을 모으고, 변수로 재사용한다(3장).
- **신호 연결**: 메트릭 → 트레이스 → 로그로 한 화면에서 점프한다(3장).
  관측가능성 시리즈에서 말한 그 correlation.
- **알림**: 규칙을 평가해 contact point로 보낸다(4장).
- **Observability as Code**: 대시보드를 코드처럼 Git으로 관리한다(6장).

## 1.5 정리

- Grafana는 **저장소가 아니라 시각화·알림 레이어** — 메타데이터만 저장.
- 170+ 데이터소스를 붙이는 **중립적 단일 창**.
- Kibana fork(2014), 최신 Grafana 13.

다음 장에서는 그 "쿼리만 위임"이 어떻게 동작하는지 — 데이터소스와 query proxy를 본다.
