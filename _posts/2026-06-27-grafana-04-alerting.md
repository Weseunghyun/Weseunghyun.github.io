---
title: "Grafana 완전 정복 4장 — Unified Alerting"
date: 2026-06-27 23:55:00 +0900
categories: ["Grafana", "Grafana 완전 정복"]
tags: ["grafana", "운영"]
---
Grafana의 알림은 **Unified Alerting**이다(8.0~).
Grafana와 Prometheus 알림을 한 모델로 통합했고, Prometheus Alertmanager 위에 지어졌다.
("무엇을 어떻게 알릴지"의 철학은 관측가능성 시리즈 9장 — 여기선 Grafana 제품의 동작.)

## 4.1 alert rule

규칙은 네 가지로 구성된다.

1. **쿼리**: 평가할 데이터(Grafana-managed는 여러 데이터소스 동시 가능).
2. **조건(threshold)**: 임계값.
3. **평가 일정**: 주기 + pending period(`for`).
4. **커스터마이즈**: 라벨·annotation·NoData/Error 처리·라우팅.

한 규칙이 라벨셋(시계열)마다 **독립 instance**를 만든다(multi-dimensional).
규칙은 Grafana DB에 저장하는 **Grafana-managed**와, 데이터소스(Prometheus/Mimir/Loki)에 저장하는 **data source-managed**로 나뉜다.
신규는 Grafana-managed가 기본이다.

## 4.2 평가와 상태

- 규칙은 evaluation group에 속하고, group이 평가 주기를 가진다.
- **pending period(`for`)**: 조건이 맞은 뒤 fire까지 유지돼야 하는 시간.
- instance 상태: **Normal · Pending · Alerting · NoData · Error**.
  - 조건 충족 → Pending → (`for` 동안 유지되면) Alerting.
  - 데이터가 0개면 NoData, 쿼리 실패면 Error.

## 4.3 notification 라우팅

알림이 사람에게 닿는 경로는 **notification policy 트리**가 정한다.

- **notification policy**: 라벨 매칭(트리)으로 어느 contact point로 보낼지 결정.
  기본은 가장 깊은 매칭 자식만 처리.
- **contact point**: 보낼 곳(email·Slack·PagerDuty·webhook·OnCall 등).
  한 contact point에 여러 통합을 묶을 수 있다.
- **grouping**: 비슷한 알림을 묶는다(Group wait 30s·interval 5m·repeat 4h 기본).
- **mute timing**(정기 무음: 주말) vs **silence**(일회성 무음: 점검).
  둘 다 알림 생성만 막고 규칙 평가는 안 막는다.

## 4.4 🔴 흔한 함정

- **NoData/Error 처리 오설정**: 기본은 NoData→NoData·Error→Error라, 데이터소스 일시 장애에 의도치 않게 발화한다 → "Keep last state" 고려.
- **notification policy 매칭 누락**: 라벨 매처 오타 → default policy로 빠져 엉뚱한 contact point로.
- **grouping 미설정**: 인스턴스마다 개별 알림 → 폭주.
- **mute timing 미상속**: 부모에 걸어도 자식엔 적용 안 됨 → 각 레벨 개별 설정.

## 4.5 정리

- **Unified Alerting**: alert rule(쿼리+조건+`for`) → 상태(Normal/Pending/Alerting/NoData/Error).
- **notification policy 트리** → contact point, grouping·mute·silence.
- 함정: NoData/Error 처리·매칭 누락·grouping 미설정.

다음 장에서는 인증·권한·고가용성을 본다.
