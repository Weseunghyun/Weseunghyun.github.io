---
title: "Tempo 완전 정복 5장 — 트레이스에서 메트릭 뽑기"
date: 2026-06-27 23:54:00 +0900
categories: ["Tempo", "Tempo 완전 정복"]
tags: ["tempo", "핵심"]
---
트레이스에는 메트릭을 만들 재료가 가득하다 — 요청 수, 지연, 에러.
Tempo는 두 방법으로 트레이스에서 메트릭을 뽑는다.
미리 만들어두는 **metrics-generator**와, 쿼리 시점에 만드는 **TraceQL metrics**다.

## 5.1 metrics-generator — 미리 만들기

**metrics-generator**는 들어오는 span에서 메트릭을 사전 생성한다.

- **span-metrics(RED)**: service·operation·status별로 span의 count·duration을 집계 → RED 메트릭(Rate·Errors·Duration).
  관측가능성 시리즈 9장의 그 RED다.
- **service graph**: 부모-자식 span 관계에서 service 간 호출 그래프(요청수·지연)를 만든다.

이 메트릭은 내장 Prometheus Agent가 **remote write로 Prometheus/Mimir에** 보낸다.
즉 "트레이스 → 메트릭"을 자동으로 만들어 메트릭 백엔드에 쌓는 것이다.

## 5.2 TraceQL metrics — 쿼리 시점에

**TraceQL metrics**는 TraceQL 결과에 함수를 적용해 **쿼리 시점에** 시계열을 만든다.

```
{ } | rate() by (resource.service.name)
{ span:duration > 1s } | quantile_over_time(0.99, ...) 
```

미리 집계해두지 않아도 ad-hoc으로 "이 조건의 trace 비율"을 그래프로 볼 수 있다.

- **상태**: 오래 실험적이었다가 **Tempo 3.0에서 GA(정식)** 가 됐다(2.x는 experimental).

## 5.3 🔴 카디널리티 주의

여기에도 카디널리티 함정이 있다.
metrics-generator는 켜는 dimension(차원)마다 메트릭 시계열이 늘어난다.

> **동적 span 이름**(`/users/123` 같은 path)이나 **고유 attribute**를 메트릭 label로 노출하면 시계열이 폭발한다.

이건 Prometheus·Loki에서 본 카디널리티 문제와 똑같다.
3.0은 label별 카디널리티 제한, span 이름 정규화(DRAIN) 같은 완화책을 넣었다.

## 5.4 정리

- **metrics-generator**: span에서 RED·service graph를 사전 생성 → Prometheus/Mimir로 remote write.
- **TraceQL metrics**: 쿼리 시점에 ad-hoc 시계열(3.0 GA).
- 둘 다 **동적 값을 메트릭 label로 노출하면 카디널리티 폭발** — 메트릭과 같은 원리.

다음 장에서 운영 함정을 모으고 면접용으로 정리한다.
