---
title: "OpenTelemetry 완전 정복 1장 — 한 번 계측, 어디로든"
date: 2026-06-27 23:58:00 +0900
categories: ["OpenTelemetry", "OpenTelemetry 완전 정복"]
tags: ["opentelemetry", "입문"]
---
## 1.1 무엇을 푸는가

0장에서 본 그 문제 — 벤더 종속 — 를 OpenTelemetry는 계측 표준화로 푼다.

![](/assets/img/posts/opentelemetry-01-opentelemetry란/otel-instrument-once.svg)

코드를 OTel 표준(전송은 **OTLP**, 이름은 **semantic conventions**)으로 한 번만 계측하면, 보내는 백엔드는 설정으로 바꾼다.
계측은 OTel, 보내는 곳은 자유 — 이게 핵심이다.

## 1.2 어떻게 표준이 됐나

OpenTelemetry는 2019년, 두 경쟁 프로젝트가 합쳐져 태어났다.
추적 API를 다루던 **OpenTracing**과, 추적+메트릭을 다루던 **OpenCensus**가 따로 노는 게 문제였는데, 그걸 병합한 것이다.
OpenTracing의 깔끔한 API/SDK 분리에 OpenCensus의 신호 폭을 더했다.

이후 metrics·logs까지 포괄하며 관측 데이터의 공통 언어가 됐고, **2026년 5월 CNCF를 졸업(Graduated)** 했다.
쿠버네티스 다음으로 활발한 CNCF 프로젝트라는 평가를 받는다.
그래서 관측가능성을 새로 도입한다면 계측은 OTel로 하는 게 사실상 기본값이다.

## 1.3 무엇으로 구성되나

| 구성 | 역할 |
|---|---|
| **API / SDK** | 앱에서 데이터를 만드는 표준 인터페이스(API) + 처리·전송 구현(SDK), 언어별 |
| **OTLP** | 데이터를 보내는 표준 프로토콜 (gRPC 4317 / HTTP 4318) |
| **Semantic conventions** | `service.name` 같은 표준 이름 규약 |
| **Collector** | 받아서 가공·라우팅하는 중계 파이프라인 |
| **계측(instrumentation)** | auto·manual·eBPF로 신호 생성 |

관측가능성 시리즈에서 "신호를 연결하려면 공통 이름(service.name)이 필요하다"고 했는데, 그 공통 이름이 **semantic conventions**(5장)다.
그리고 그걸 실어 나르는 규격이 **OTLP**(5장)다.

## 1.4 신호별 성숙도 (2026-06)

OTel은 세 신호를 다루지만 성숙도가 조금씩 다르다.

- **Traces**: API/SDK/프로토콜 전부 안정(stable).
- **Metrics**: 대체로 안정.
- **Logs**: 안정이긴 하나 가장 늦게 표준화됐고, "기존 로깅을 OTel로 **연결(bridge)**"하는 접근이라 결이 다르다(2장).
- **Profiles**(4번째 신호): 아직 개발 중(stable 아님).
  "이미 안정화됐다"는 식의 설명은 사실이 아니다.

## 1.5 정리

- OpenTelemetry는 **벤더 종속을 푸는** 계측 표준 — "한 번 계측, 어디로든".
- OpenTracing + OpenCensus 병합(2019), **2026-05 CNCF 졸업**.
- 구성: **API/SDK · OTLP · semantic conventions · Collector**.
- traces/metrics/logs는 안정, profiles는 아직 개발 중.

다음 장에서는 왜 API와 SDK를 나눴고, 신호별로 어떻게 동작하는지 본다.
