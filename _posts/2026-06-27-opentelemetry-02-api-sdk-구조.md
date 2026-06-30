---
title: "OpenTelemetry 완전 정복 2장 — API/SDK와 신호별 구조"
date: 2026-06-27 23:57:00 +0900
categories: ["OpenTelemetry", "OpenTelemetry 완전 정복"]
tags: ["opentelemetry", "핵심"]
---
## 2.1 🔴 왜 API와 SDK를 나눴나

OpenTelemetry의 가장 중요한 설계 결정은 **API와 SDK를 분리**한 것이다.

- **라이브러리·프레임워크는 API에만 의존**한다.
  어떤 OTel 구현이 최종 앱에 깔릴지 몰라도 된다.
- **앱(소유자)이 SDK를 설치**하고 exporter 등을 설정한다.

왜 이게 중요할까.
SDK가 안 깔려 있으면 API 호출은 **아무 동작도 안 하는(no-op)** 빈 구현으로 떨어진다 — 오버헤드가 거의 0이다.
덕분에 어떤 라이브러리가 OTel로 계측돼 있어도, 그걸 쓰는 앱이 OTel을 안 써도 멀쩡히 돌아간다.
라이브러리 작성자가 마음 놓고 계측을 넣을 수 있는 것이다.

## 2.2 신호별 구조 — 같은 모양

세 신호는 거의 같은 모양이다.

> `XxxProvider` → `Xxx` → 데이터, 그리고 **Processor + Exporter** 파이프라인.

**Tracing**: `TracerProvider` → `Tracer` → `Span`.
- **Span**에는 이름·시작/종료 시각·attributes·events·status가 담긴다.
- **SpanProcessor**가 export 전 처리를 한다.
  `BatchSpanProcessor`(배치 모아 전송, 운영 권장)와 `SimpleSpanProcessor`(즉시, 디버그용)가 있다.
- 🔴 **함정 하나** — BatchSpanProcessor는 기본 5초마다 모아 보낸다.
  짧게 끝나는 함수가 그 전에 종료되면 모은 span이 사라진다(6장).

**Metrics**: `MeterProvider` → `Meter` → `Instrument`.
- instrument는 Counter·UpDownCounter·Histogram·Gauge(+ 비동기 관측형).
- **View**로 집계 방식·이름·attribute를 커스터마이즈한다 — attribute를 필터해 카디널리티를 통제하는 데 쓴다.
- **temporality**: 누적(Cumulative, Prometheus 스타일) vs 증분(Delta) 중 백엔드에 맞춰 고른다.

**Logs**: `LoggerProvider` → `Logger`.
- 결이 다르다 — 기존 로깅 생태계가 워낙 크니, 새 로깅 API를 강요하지 않고 **기존 로그를 OTel로 연결(bridge)** 한다.
  그래서 공식 명칭이 **Logs Bridge API**다.

## 2.3 Resource — 누가 보냈나

모든 신호에는 **Resource**가 붙는다.
"이 데이터를 누가 만들었나"의 메타데이터다 — `service.name`·`service.version`·호스트·컨테이너 정보.
환경에서 자동으로 탐지(detector)하거나 `OTEL_RESOURCE_ATTRIBUTES`로 준다.
`service.name`이 없으면 백엔드에서 누가 보낸 건지 식별이 안 되니 사실상 필수다(6장).

## 2.4 Context와 전파

서비스를 넘나드는 trace를 잇는 건 **Context**다.
현재 span 정보와 baggage를 담는 객체로, 보통 스레드에 묶여 자동으로 따라다닌다.
서비스 경계를 넘을 땐 **Propagator**가 이걸 HTTP 헤더(W3C Trace Context 등)로 직렬화·복원한다.
이 context가 async·큐 경계에서 끊기는 게 트레이싱의 단골 사고다(6장).

## 2.5 설정 — 환경변수와 선언적 설정

- **환경변수**(`OTEL_*`)가 가장 간단하다 — `OTEL_SERVICE_NAME`·`OTEL_EXPORTER_OTLP_ENDPOINT`·`OTEL_TRACES_SAMPLER` 등(7장).
- 2026년엔 파이프라인을 단일 YAML로 정의하는 **declarative config**가 안정화됐다.

## 2.6 정리

- **API/SDK 분리** — 라이브러리는 API만, 앱이 SDK 선택.
  SDK 없으면 no-op(오버헤드 0).
- 신호별로 `Provider → ... → Processor → Exporter` 같은 모양.
  **BatchSpanProcessor 5초** 주의.
- **Resource**(service.name 필수)와 **Context 전파**가 신호를 묶는다.

다음 장에서는 실제로 앱에 OTel을 붙이는 방법 — 자동·수동·eBPF 계측이다.
