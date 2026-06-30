---
title: "OpenTelemetry 완전 정복 3장 — 계측, auto·manual·eBPF"
date: 2026-06-27 23:56:00 +0900
categories: ["OpenTelemetry", "OpenTelemetry 완전 정복"]
tags: ["opentelemetry", "핵심"]
---
앱에 OpenTelemetry를 붙이는 길은 셋이다.
코드를 안 고치는 **자동(zero-code)**, API로 직접 짜는 **수동**, 커널에서 잡는 **eBPF**.
보통 자동 + 수동을 함께 쓴다.

## 3.1 🔴 자동 계측 — 언어별 메커니즘

소스를 안 고치고 붙이는 방식이다.
공통적으로 들어오는 요청·나가는 HTTP·DB 호출 같은 라이브러리 경계를 자동으로 잡는다.
다만 **언어마다 메커니즘이 다르다**.

| 언어 | 메커니즘 | 부착 |
|---|---|---|
| **Java** | 바이트코드 동적 주입 | `-javaagent:opentelemetry-javaagent.jar` |
| **Python** | monkey-patching | `opentelemetry-instrument python app.py` |
| **Node.js** | 모듈 preload | `node --require @opentelemetry/auto-instrumentations-node/register` |
| **.NET** | profiler + startup hook | 설치 스크립트 |
| **Go** | (컴파일 언어라 주입 불가) **eBPF** | 별도 에이전트 |

Java는 JVM에 에이전트 jar를 끼우면 바이트코드를 조작해 계측을 주입한다.
Python·Node는 런타임에 함수를 바꿔치기(monkey-patching)한다.
Go는 컴파일된 바이너리라 이 방식이 안 돼서 eBPF로 우회한다(아래).

## 3.2 수동 계측

자동이 못 잡는 비즈니스 로직 구간은 API로 직접 span·metric을 추가한다.

```
Span span = tracer.spanBuilder("checkout").startSpan();
try (Scope s = span.makeCurrent()) {
    span.setAttribute("order.id", id);
    // ... 작업 ...
} finally {
    span.end();
}
```

자동과 수동은 같은 전역 인스턴스를 공유하니, 자동으로 만든 span 안에 수동 span이 자연스럽게 자식으로 붙는다.
"자동으로 큰 틀, 수동으로 중요한 구간 보강"이 표준이다.

## 3.3 instrumentation 라이브러리

프레임워크에 OTel 지원이 내장돼 있지 않으면, 그 프레임워크(Spring·Express·Flask)를 감싸 계측해주는 **instrumentation 라이브러리**가 있다.
메인테이너가 OpenTelemetry 레지스트리에 등록해두면 사용자가 찾아 쓴다.

## 3.4 eBPF / OBI — 코드 없이 커널에서

요즘 흐름은 **코드를 전혀 안 건드리는** 방식이다.
**eBPF**라는 리눅스 커널 기술로, 프로세스 밖에서 프로토콜 레벨(HTTP·gRPC·SQL 등)을 잡는다.

Grafana의 Beyla가 OpenTelemetry에 기증돼 **OpenTelemetry eBPF Instrumentation(OBI)** 이 됐다.

- **장점**: 언어 무관, 컴파일 바이너리도 자동, 앱 성능 영향 최소.
- **한계**: 커널 레벨이라 **앱 내부 비즈니스 로직 span의 깊이는 제한적**이다.
  그래서 "eBPF와 SDK 둘 다 필요"라는 게 현실적 결론이다.

## 3.5 쿠버네티스 자동 주입

쿠버네티스에서는 **OpenTelemetry Operator**가 자동 계측을 파드에 주입한다.
Instrumentation이라는 설정(CRD)을 만들고 파드에 어노테이션을 달면, Operator가 초기화 컨테이너로 에이전트를 앱에 복사해 넣는다.
앱 이미지를 안 고치고도 계측이 붙는다.

## 3.6 🔴 흔한 실수

- **`service.name` 누락** — 백엔드에서 누가 보낸 건지 식별 불가.
  가장 흔하다.
- **flush 전 종료** — 짧은 함수가 BatchSpanProcessor의 5초 전에 끝나면 span 유실(6장).
- **고카디널리티 attribute** — user_id·전체 URL을 모든 span에.
  특히 **span 이름에 고유값을 넣으면** 백엔드 인덱스가 폭발한다 — 고유값은 attribute로.
- **async/큐 경계 context 끊김** — 수동 전파가 필요하다(6장).

## 3.7 정리

- **자동(zero-code)**: Java=바이트코드·Python/Node=monkeypatch·Go=eBPF.
- **수동**: API로 직접 span — 자동과 함께 쓰는 게 표준.
- **eBPF(OBI)**: 코드 없이 커널에서 — 언어 무관, 단 깊이 제한.
- 실수 1순위는 **service.name 누락**.

다음 장에서는 계측이 보낸 데이터를 받는 쪽 — Collector다.
