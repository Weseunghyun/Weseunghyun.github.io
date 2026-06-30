---
title: "OpenTelemetry 완전 정복 5장 — OTLP와 semantic conventions, 표준의 두 축"
date: 2026-06-27 23:54:00 +0900
categories: ["OpenTelemetry", "OpenTelemetry 완전 정복"]
tags: ["opentelemetry", "핵심"]
---
OpenTelemetry가 "한 번 계측, 어디로든"을 가능하게 하는 건 두 표준 덕분이다.
데이터를 **어떻게 실어 나르나**(OTLP), 그리고 **무엇이라 부르나**(semantic conventions).

## 5.1 OTLP — 전송 규격

**OTLP(OpenTelemetry Protocol)** 는 telemetry를 실어 나르는 표준 와이어 프로토콜이다.

- 전송: **gRPC(포트 4317)** 또는 **HTTP(4318)**.
  HTTP는 protobuf나 JSON.
- HTTP 경로는 고정 — `/v1/traces`·`/v1/metrics`·`/v1/logs`.
- trace·metric·log 신호에서 **안정(stable)** 이다.

## 5.2 데이터 모델 — Resource → Scope → Span

OTLP가 나르는 데이터는 계층 구조다.

```
ResourceSpans → ScopeSpans → Span
```

- **Resource**: 누가 보냈나(`service.name` 등).
- **Scope**: 어느 계측 라이브러리가 만들었나(이름·버전).
- **Span**: 실제 작업 단위.

metric·log도 같은 모양(`ResourceMetrics → ScopeMetrics → Metric`)이다.
여기에 `schema_url`이 붙어 어떤 semantic conventions 버전을 따르는지 표시한다.

## 5.3 🔴 endpoint 경로 함정

실무에서 자주 밟는 함정 하나.
OTLP endpoint를 환경변수로 줄 때, **base 변수와 신호별 변수의 동작이 다르다**.

- `OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4318` → SDK가 `/v1/traces`를 **자동으로 붙인다**.
- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://collector:4318` → 신호별 변수는 **있는 그대로** 쓴다 → `/`로 보내져 404.

신호별 변수를 쓸 땐 경로(`/v1/traces`)까지 직접 적어야 한다.

## 5.4 semantic conventions — 표준 이름

데이터를 보내도 이름이 제각각이면 백엔드가 묶지 못한다.
한쪽은 `http.method`, 다른 쪽은 `method`로 부르면 같은 개념인지 알 수 없다.

**semantic conventions**는 이 이름을 표준화한 규약이다.
모두가 서비스를 `service.name`으로, HTTP 메서드를 `http.request.method`로 부른다.
이 공통 이름 덕에 메트릭·로그·트레이스가 서로 묶이고(관측가능성 시리즈의 신호 연결), 백엔드가 데이터를 이해한다.

대표적인 표준 attribute는 이렇다.

| 분류 | 예 |
|---|---|
| 리소스 | `service.name`(필수)·`service.version`·`deployment.environment.name` |
| HTTP | `http.request.method`·`http.response.status_code`·`url.full` |
| DB | `db.system`·`db.query.text` |

## 5.5 🔴 stability — 이름이 바뀐다

semantic conventions에는 안정(stable)과 실험(experimental)이 있고, **이름이 바뀌기도 한다.**

대표적으로 HTTP 규약이 안정화되며 대거 rename됐다.

- `http.method` → `http.request.method`
- `http.status_code` → `http.response.status_code`
- `http.url` → `url.full`

그래서 옛 이름을 기대하던 대시보드·알림이 깨질 수 있다.
`OTEL_SEMCONV_STABILITY_OPT_IN=http/dup`으로 옛·새 이름을 같이 내보내며 단계적으로 전환한다.

## 5.6 정리

- **OTLP**: 전송 표준(gRPC 4317·HTTP 4318), 데이터는 **Resource→Scope→Span** 계층.
- 신호별 endpoint 변수는 경로를 **자동으로 안 붙인다**(404 함정).
- **semantic conventions**: 표준 이름이 신호 연결·백엔드 호환의 전제.
- 규약은 진화한다 — **HTTP rename** 같은 변경에 대시보드가 깨질 수 있으니 단계 전환.

다음 장에서는 OTel을 쓰다 실제로 어디서 터지는지 모아 본다.
