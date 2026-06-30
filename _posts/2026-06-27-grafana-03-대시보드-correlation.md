---
title: "Grafana 완전 정복 3장 — 대시보드·변수·변환·신호 연결"
date: 2026-06-27 23:56:00 +0900
categories: ["Grafana", "Grafana 완전 정복"]
tags: ["grafana", "핵심"]
---
## 3.1 대시보드 모델

- **패널(panel)**: 쿼리 + 시각화 한 단위.
  time series·stat·gauge·table·logs·traces 등 120+ 타입.
- **변수(variable)**: 드롭다운으로 같은 대시보드를 여러 대상에 재사용.
  query·custom·interval·datasource 등 여러 종류가 있고, 변수끼리 연쇄(chained)도 된다.
- **library panel**: 여러 대시보드가 공유하는 패널(한 곳 고치면 전파).
- 대시보드는 JSON으로 직렬화되어 DB나 Git에 저장된다(6장).

## 3.2 🔴 transformation — 프론트에서 가공

**transformation**은 쿼리 결과를 **시각화 전에 브라우저에서** 가공하는 기능이다.
데이터소스를 건드리지 않고, 여러 변환을 순서대로 체인한다.

- Join by field(여러 쿼리 결과 합치기), Organize(이름변경·숨김), Add field from calculation(계산 필드), Group by, Filter…
- **순서가 결과에 영향**을 준다 — 앞 변환의 출력이 뒤 변환의 입력이다.

"쿼리로는 안 나오는 표/계산"을 데이터소스 수정 없이 만들 때 쓴다.

## 3.3 🔴 신호 연결 (correlation)

관측가능성 시리즈에서 "연결이 진짜 가치"라고 했다.
Grafana에서 그게 실제로 동작하는 지점이 여기다.

| 메커니즘 | 동작 |
|---|---|
| **exemplar** | 메트릭 그래프의 점 → 그 trace로 점프 |
| **trace → logs** | Tempo span → Loki의 매칭 로그 |
| **trace → metrics** | Tempo span → Prometheus 메트릭 |
| **data links** | 패널 값 → 다른 대시보드·외부 URL |

이 설정 덕에 "메트릭 스파이크 → 그 trace → 그 로그"를 **한 Grafana 화면 안에서** 클릭으로 따라간다.
백엔드 쪽 준비(exemplar 노출, trace_id를 로그에 박기)는 Prometheus·Loki·Tempo 시리즈에서 본 그대로다.

## 3.4 Explore와 Drilldown

- **Explore**: 대시보드 밖에서 임시 쿼리·디버깅하는 단일 화면.
- **Drilldown 앱**(구 Explore apps): 쿼리를 직접 안 짜도 메트릭·로그·트레이스를 탐색하는 UI.

## 3.5 정리

- 대시보드 = **패널 + 변수**(+ library panel), JSON으로 직렬화.
- **transformation**: 프론트에서 쿼리 결과 가공(순서 중요).
- **correlation**(exemplar·trace→logs·trace→metrics)으로 한 화면에서 신호 간 점프.

다음 장에서는 Grafana의 알림(unified alerting)을 본다.
