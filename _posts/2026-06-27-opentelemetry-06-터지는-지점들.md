---
title: "OpenTelemetry 완전 정복 6장 — 터지는 지점들"
date: 2026-06-27 23:53:00 +0900
categories: ["OpenTelemetry", "OpenTelemetry 완전 정복"]
tags: ["opentelemetry", "운영"]
---
OpenTelemetry 사고는 크게 두 갈래다.
데이터가 **조용히 사라지거나**(끊김·유실), **너무 많아 터지거나**(카디널리티·OOM).
거기에 버전·이름 어긋남이 더해진다.

## 6.1 🔴 context 끊김 → trace 단절

OTel은 현재 context를 보통 **스레드에 묶어**둔다.
그래서 작업이 다른 스레드·비동기 콜백·**메시지 큐**로 넘어가면 context가 자동으로 안 따라가, trace가 끊기고 부모 없는 span이 생긴다.

- **메시지 큐**: 넣을 때 헤더에 context를 싣고, 꺼낼 때 다시 읽어야 한다.
  안 하면 큐 경계마다 끊긴다.
- 일부 언어·라이브러리는 자동 계측이 async 경계를 못 잡아 수동 전파가 필요하다.

## 6.2 🔴 flush 전 종료 → span 유실

`BatchSpanProcessor`는 기본 5초마다 모아 보낸다(2장).
짧게 끝나는 함수(서버리스·짧은 배치)가 그 5초 전에 종료되면, 모아둔 span이 안 보내지고 사라진다.
종료 전에 강제로 내보내야(flush) 한다.

## 6.3 🔴 attribute 카디널리티 폭발

user_id·전체 URL을 모든 span/metric attribute에 붙이면 백엔드 저장·쿼리 비용이 폭증한다.
특히 **span 이름에 고유값을 넣지 말아야** 한다 — `/users/123`처럼 동적 값이 이름에 들어가면 백엔드 인덱스가 폭발한다.
고유값은 attribute로, 메트릭은 View로 attribute를 걸러낸다.
메트릭으로 가면 이건 곧 Prometheus 카디널리티 문제와 같다.

## 6.4 Collector OOM과 tail sampling

- Collector가 들어오는 데이터를 다 못 받으면 메모리가 터진다.
  **memory_limiter를 파이프라인 맨 앞**에 둬야 한도 초과분을 거부해 살아남는다(4장).
- **tail sampling**은 요청 끝날 때까지 span을 메모리에 들고 있어 더 위험하다.
  게다가 같은 trace가 여러 Collector로 흩어지면 잘못된 샘플링이 되니 **trace ID로 같은 Collector에 모아야** 한다(4장).

## 6.5 service.name 누락

`service.name`이 없으면 대부분의 백엔드에서 **누가 보낸 건지 식별이 안 된다.**
가장 흔하고 가장 기본적인 실수다(2·3장).

## 6.6 버전·이름 어긋남

- **API/SDK 버전 불일치**: 라이브러리가 새 API를 쓰는데 앱의 SDK가 옛 버전이면 어긋난다.
- **semconv 변경**: HTTP attribute가 `http.method` → `http.request.method`로 바뀌며 옛 이름을 쓰던 대시보드·알림이 깨진다.
  `http/dup`으로 단계 전환(5장).

## 6.7 OTLP endpoint 경로

신호별 endpoint 변수는 `/v1/traces`를 자동으로 붙이지 않는다 — base 변수와 헷갈려 경로를 빼먹으면 404다(5장).

## 6.8 정리

> 적게 보내 잃거나(context 끊김·flush·service.name), 많이 보내 터지거나(attribute 카디널리티·Collector OOM), 버전·경로가 어긋난다.

데이터가 안 보일 때 점검 순서: **context · flush · service.name · endpoint**.

다음 장에서 직접 계측해보고 면접용으로 정리한다.
