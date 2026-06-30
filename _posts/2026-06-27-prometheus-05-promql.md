---
title: "Prometheus 완전 정복 5장 — PromQL, 기초부터 vector matching까지"
date: 2026-06-27 23:54:00 +0900
categories: ["Prometheus", "Prometheus 완전 정복"]
tags: ["prometheus", "핵심"]
---
저장된 메트릭에 질문하는 언어가 **PromQL**이다.
처음 보면 낯설지만, 몇 가지 핵심과 함정만 잡으면 대부분 풀린다.

## 5.1 instant vector vs range vector

PromQL의 가장 기본 구분이다.

- **instant vector**: 각 시계열의 "지금 이 순간 값 하나".
  예: `http_requests_total`
- **range vector**: 각 시계열의 "지난 5분치 값들".
  예: `http_requests_total[5m]`

`rate()` 같은 함수는 range vector(구간)를 받아 instant vector(순간값)를 만든다.
이 변환을 이해하면 PromQL의 절반은 잡은 셈이다.

## 5.2 rate() — 카운터의 속도

counter는 누적값이라 절대값(`1027`)은 의미가 약하다.
중요한 건 **얼마나 빨리 늘고 있나**다.
그래서 `rate()`로 초당 증가 속도를 본다.

```
rate(http_requests_total[5m])   # 최근 5분 기준 초당 요청 수
```

`rate()`는 카운터가 재시작으로 0이 되는 것도 자동 보정한다.
비슷한 `irate()`도 있는데, 마지막 두 점만 봐서 그래프가 들쭉날쭉하다 — **알림에는 `rate()`를 쓴다**(공식 권장).

## 5.3 집계 — by와 without

여러 시계열을 합칠 땐 `sum`·`avg`·`max` 같은 집계에 `by`/`without`를 붙인다.

- `sum by (job) (...)`: 나열한 `job`만 남기고 나머지로는 합친다.
- `sum without (instance) (...)`: 나열한 `instance`만 빼고 나머지는 유지한다.

## 5.4 🔴 PromQL 4대 함정

초보가 거의 반드시 한 번은 밟는다.

**함정 1 — rate 먼저, sum 나중.**
```
sum(rate(http_requests_total[5m]))     # 맞음
rate(sum(http_requests_total)[5m])     # 틀림
```
먼저 `sum`하면 개별 서버 재시작으로 카운터가 떨어진 걸 감지 못해 값이 망가진다.
반드시 **각자의 속도를 rate로 먼저** 구한 뒤 합친다.

**함정 2 — 구간은 스크레이프 주기의 최소 2배.**
`rate`는 구간에 점이 2개 이상 있어야 계산된다.
15초마다 긁는데 `[15s]`로 잡으면 점이 1개뿐일 수 있어 결과가 빈다.

**함정 3 — increase()가 정수가 아니다.**
정수만 세는 카운터인데 `increase()`가 `2.58` 같은 소수를 낼 때가 있다.
버그가 아니라 구간 경계까지 기울기를 이어붙여 추정하기 때문이다.

**함정 4 — irate()는 알림에 부적합.** (위 5.2)

## 5.5 분위수 — histogram_quantile

2장의 histogram으로 p99를 구하는 방법이다.

```
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

순서가 중요하다 — bucket은 카운터라 **`rate()`를 먼저** 씌우고, 집계할 땐 **`le`(구간 경계) 라벨을 반드시 남겨야** 한다(`by (le)`).
`le`를 빠뜨리면 분위수 계산이 깨진다.
이게 5.4의 "집계 시 라벨 손실" 함정의 대표 사례다.

bucket 내부는 직선으로 보간하므로 경계가 거칠면 부정확하다 — native histogram(2장)은 이걸 더 정밀하게 푼다.

## 5.6 🔴 vector matching — 서로 다른 메트릭 엮기

에러율(에러÷전체) 같은 건 **두 메트릭을 나눠야** 나온다.
이때 PromQL은 양쪽에서 **같은 라벨 조합을 찾아 짝짓는다**.

- 라벨이 정확히 같으면 그냥 `a / b`.
- 일부 라벨만으로 짝지으려면 `on(...)`(이 라벨만)이나 `ignoring(...)`(이 라벨만 빼고).

한쪽은 여러 개, 다른 쪽은 하나일 때(예: code별 에러 ÷ 전체 요청)는 `group_left`로 "왼쪽이 여러 개(many)"임을 알린다.

```
errors:rate5m / ignoring(code) group_left requests:rate5m
```

여기서 자주 보는 에러가 `many-to-many matching not allowed`다.
"하나여야 할 쪽(one)이 유일하지 않다"는 뜻이고, 그쪽을 집계해 유일하게 만들거나 매칭 라벨을 더 좁혀 해결한다.

## 5.7 정리

- **instant vs range vector**, `rate()`는 range를 받아 속도를 낸다.
- 집계는 `by`/`without`, **rate 먼저 sum 나중**, 알림엔 rate.
- 분위수는 **`histogram_quantile` + `rate` + `by (le)`**.
- 서로 다른 메트릭은 **vector matching**(`on`/`ignoring`/`group_left`).

다음 장에서는 이 시계열이 너무 많아질 때 벌어지는 일 — 카디널리티와, 그걸 넘기 위한 장기저장이다.
