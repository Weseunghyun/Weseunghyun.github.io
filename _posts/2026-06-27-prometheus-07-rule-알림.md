---
title: "Prometheus 완전 정복 7장 — rule과 알림"
date: 2026-06-27 23:52:00 +0900
categories: ["Prometheus", "Prometheus 완전 정복"]
tags: ["prometheus", "운영"]
---
Prometheus는 주기적으로 두 종류의 규칙(rule)을 평가한다.
비싼 쿼리를 미리 계산해두는 **recording rule**, 그리고 조건이 맞으면 알림을 발생시키는 **alerting rule**이다.
알림의 발송 자체는 별도 컴포넌트인 Alertmanager가 맡는다.

## 7.1 recording rule — 미리 계산해두기

자주 쓰거나 비싼 PromQL을 매번 실행하면 느리다.
**recording rule**은 그 결과를 미리 계산해 **새 시계열로 저장**해둔다.
대시보드나 알림이 그 새 시계열을 읽으면 빠르다.

```yaml
groups:
  - name: http
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
```

이름 규약이 있다 — `level:metric:operation` 형태(`job:http_requests:rate5m`).
콜론(`:`)이 규칙 전용인 게 이래서다(2장).

**순서가 중요하다.**
같은 그룹 안의 규칙은 순차로, 같은 시각에 평가된다 — 그래서 앞 규칙 결과를 뒤 규칙이 참조할 수 있다.
서로 다른 그룹은 병렬로 평가되니, 의존 관계가 있으면 같은 그룹에 둬야 한다.

## 7.2 alerting rule — 조건과 for

알림 규칙은 조건이 맞으면 알림을 발생시킨다.

```yaml
- alert: HighErrorRate
  expr: job:http_errors:rate5m / job:http_requests:rate5m > 0.05
  for: 10m
  labels: { severity: page }
  annotations:
    summary: "{{ $labels.job }} 에러율 5% 초과"
```

핵심은 **`for`** 다.
조건이 맞자마자 알리는 게 아니라, `for` 기간(여기선 10분) 동안 **계속** 맞아야 발화한다.
조건이 처음 맞으면 **pending**, 10분 내내 유지되면 **firing**이 되어 Alertmanager로 간다.
이 `for`가 일시적 스파이크로 인한 거짓 알림을 막는다.

## 7.3 Alertmanager — 알림을 사람에게

Prometheus는 firing 알림을 **Alertmanager**로 보내고, Alertmanager가 사람에게 닿기까지를 처리한다.

| 단계 | 하는 일 |
|---|---|
| dedupe | 중복 제거 |
| group | 비슷한 알림을 하나로 묶기 |
| route | 어느 채널(Slack·PagerDuty)로 |
| silence | 일정 시간 무음(점검 중) |
| inhibit | 상위 알림이 떴으면 하위 억제 |

특히 group과 inhibit가 중요하다.
대형 장애가 나면 알림이 수백 개씩 쏟아지는데, 묶고 억제해서 "클러스터 전체가 죽었다" 하나로 정리한다.
무엇을 어떻게 알릴지(증상 기반·SLO·error budget)의 철학은 관측가능성 시리즈 9장에서 다뤘다 — 여기선 메커니즘만 본다.

## 7.4 정리

- **recording rule**: 비싼 쿼리를 미리 계산해 새 시계열로(같은 그룹은 순차·동일 시각).
- **alerting rule**: 조건 + **`for`**(pending → firing) — `for`가 거짓 알림을 막는다.
- 알림 발송은 **Alertmanager**(dedupe·group·route·silence·inhibit).

다음 장은 Prometheus 한 대로 감당이 안 될 때 — HA·federation·remote write·agent mode다.
