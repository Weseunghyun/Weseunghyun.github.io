---
title: "OpenTelemetry 완전 정복 4장 — Collector, 수집·가공·라우팅 허브"
date: 2026-06-27 23:55:00 +0900
categories: ["OpenTelemetry", "OpenTelemetry 완전 정복"]
tags: ["opentelemetry", "핵심"]
---
계측해서 만든 데이터를 백엔드로 바로 보낼 수도 있다.
하지만 실무에서는 보통 **Collector**를 한 번 거친다.
왜 거치는지, 어떻게 생겼는지 보자.

## 4.1 왜 Collector를 거치나

앱이 백엔드로 직접 보내면, 보내는 곳을 바꿀 때마다 앱을 재배포해야 한다.
Collector를 가운데 두면 다르다.

- 보내는 대상을 바꿔도 **앱 재배포가 필요 없다**(Collector 설정만 변경).
- 배치·압축·재시도로 전송 효율을 높인다.
- 민감정보 제거나 샘플링을 **중앙에서** 한다.
- 자격증명을 앱이 아니라 Collector가 관리한다.

## 4.2 파이프라인 — receiver·processor·exporter

![](/assets/img/posts/opentelemetry-04-collector/otel-collector-pipeline.svg)

Collector는 세 단계 파이프라인이다.

- **receiver**: 데이터를 받는다(otlp·prometheus·jaeger·zipkin·filelog 등).
- **processor**: 가공한다.
  **순서가 곧 실행 순서**다.
- **exporter**: 백엔드로 내보낸다(여러 곳 동시 가능).

여기에 두 가지가 더 있다.
**connector**는 한 파이프라인의 출력을 다른 파이프라인의 입력으로 잇는다(예: trace를 받아 메트릭을 만들어 다른 파이프라인으로).
**extension**은 데이터 경로 밖의 부가기능(헬스체크·인증)이다.

🔴 processor 순서에서 가장 중요한 건 **memory_limiter를 맨 앞에** 두는 것이다.
들어오는 데이터가 너무 많으면 한도 초과분을 거부해 Collector가 OOM으로 죽는 걸 막는다.
batch 뒤에 두면 이미 쌓인 다음이라 늦다.

## 4.3 배포 — agent vs gateway

Collector를 어디에 둘지 두 패턴이 있다.

- **agent**: 각 노드/파드 옆(DaemonSet·sidecar).
  앱에서 받아 export.
  앱 변경 없이 부담을 던다.
- **gateway**: 중앙에 한 곳.
  여러 agent에서 받아 샘플링·필터·라우팅.

흔한 조합은 **노드별 agent → 중앙 gateway → 백엔드**다.

## 4.4 🔴 tail sampling의 분산 함정

중앙 gateway에서 tail sampling(요청 끝나고 저장 결정)을 하려면 주의할 게 있다.
한 요청의 span들이 여러 Collector로 흩어지면, 각자 부분만 보고 서로 다른 결정을 내려 span이 누락된다.
그래서 **같은 trace는 같은 Collector로** 모아야 하고, 이를 위해 trace ID로 라우팅하는 load-balancing exporter를 앞단에 둔다.
(관측가능성 시리즈 10장의 그 함정이다.)

## 4.5 distribution과 OTTL

- Collector는 **core**(최소)와 **contrib**(모든 컴포넌트 번들, 거대함)로 나뉜다.
  contrib가 큰 이유는 벤더별 통합이 대거 들어있어서다.
  필요한 것만 골라 빌드하는 **ocb**(빌더)도 있다.
- processor 중 **transform**은 **OTTL**이라는 변환 언어로 데이터를 임의로 가공한다(상태 코드 바꾸기, span 이름 정규화 등).
- 2026년엔 3장의 eBPF 계측(OBI)을 Collector의 receiver로 돌리는 통합이 진행 중이다.

## 4.6 정리

- Collector를 거치면 **앱 재배포 없이** 파이프라인을 바꾸고, 샘플링·필터를 중앙화한다.
- **receiver → processor → exporter**(+connector·extension), **memory_limiter는 맨 앞**.
- 배포는 **agent**(노드 옆)·**gateway**(중앙), 흔히 둘을 조합.
- tail sampling은 **같은 trace를 같은 Collector로** 모아야 한다.

다음 장에서는 데이터를 실어 나르는 규격(OTLP)과 이름 붙이는 규약(semantic conventions)을 본다.
