---
title: "관측가능성 완전 정복 7장 — OpenTelemetry, 계측의 표준"
date: 2026-06-25 23:52:00 +0900
categories: ["Observability", "관측가능성 완전 정복"]
tags: ["observability", "핵심"]
---
1장에서 OpenTelemetry를 "지금 이 분야의 중심축"이라고 했다.
이 장에서 그 정체를 제대로 본다.

## 7.1 OpenTelemetry가 푸는 문제

OpenTelemetry(줄여서 OTel) 이전에는 문제가 있었다.
관측 도구를 바꾸려면 코드를 다시 계측해야 했다.
Datadog을 쓰다가 Grafana로 옮기려면, 앱 곳곳에 박아둔 Datadog 전용 계측 코드를 전부 걷어내고 새로 깔아야 했다.
**벤더 종속(lock-in)** 이다.

OpenTelemetry의 약속은 한 문장이다.

> 한 번 계측하면, 어디로든 보낸다 (Instrument once, send anywhere).

코드를 OpenTelemetry 표준으로 한 번만 계측해두면, 보내는 대상은 설정만 바꾸면 된다.
앱을 안 고치고 Datadog → Grafana → 자체 백엔드로 자유롭게 바꿀 수 있다.

## 7.2 어떻게 표준이 됐나

OpenTelemetry는 2019년, 두 경쟁 프로젝트가 합쳐져 태어났다.
추적 API를 다루던 **OpenTracing**과, 추적+메트릭을 다루던 **OpenCensus**가 따로 노는 게 문제였는데, 둘을 병합한 것이다.

구성 요소는 이렇다.

| 요소 | 역할 |
|---|---|
| **API / SDK** | 앱에서 데이터를 만드는 표준 인터페이스(언어별) |
| **OTLP** | 데이터를 보내는 표준 프로토콜 (gRPC 4317 / HTTP 4318 포트) |
| **Semantic conventions** | `service.name` 같은 표준 이름 규약 |
| **Collector** | 받아서 가공하고 내보내는 중계 파이프라인 |

2장에서 "신호를 연결하려면 표준 이름이 필요하다"고 했는데, 그 표준 이름이 바로 **semantic conventions**다.
모두가 서비스를 같은 키(`service.name`)로 부르니 메트릭·로그·트레이스가 묶인다.

2026년 5월, OpenTelemetry는 CNCF에서 **정식 졸업(Graduated)** 했다.
쿠버네티스 다음으로 활발한 CNCF 프로젝트라는 평가를 받는다.
그래서 관측가능성을 새로 도입한다면, 계측은 OpenTelemetry로 하는 게 사실상 기본값이 됐다.

(신호별 성숙도는 조금씩 다르다 — 추적·메트릭·로그는 안정 단계이고, 네 번째 신호인 프로파일은 2026년 6월 기준 아직 개발 중이다.)

## 7.3 Collector — 가운데 중계소

OpenTelemetry에서 실무적으로 가장 중요한 부품이 **Collector**다.
앱이 만든 데이터를 백엔드로 바로 보내지 않고, 보통 이 Collector를 한 번 거치게 한다.

Collector는 세 단계 파이프라인이다.

```
receiver(받기) → processor(가공) → exporter(내보내기)
```

- **receiver**: 여러 형식(OTLP·Jaeger·Zipkin·Prometheus 등)으로 데이터를 받는다.
- **processor**: 배치로 묶고, 민감정보를 지우고, 샘플링한다.
- **exporter**: 백엔드로 내보낸다(여러 곳으로 동시에 가능).

**왜 굳이 Collector를 거칠까.**
- 보내는 대상을 바꿀 때 앱을 재배포할 필요가 없다(Collector 설정만 바꾸면 됨).
- 배치·압축·재시도로 효율을 높인다.
- 민감정보 제거나 tail 샘플링을 중앙에서 한다.

이 Collector를 어떻게 배치하느냐(각 노드 옆이냐 중앙 한 곳이냐)는 11장 실습에서 다룬다.

## 7.4 추적 백엔드 — Jaeger vs Tempo vs Zipkin

계측해서 보낸 트레이스를 저장·조회하는 백엔드도 골라야 한다.
대표 셋을 비교하면 이렇다.

| | Zipkin | Jaeger | Grafana Tempo |
|---|---|---|---|
| 기원 | 트위터, 2012 (원조) | 우버, 현 CNCF | Grafana, 2021 |
| 색인 | 저장 시 색인 | 백엔드에 전체 색인(ES 등) | **외부 색인 DB 없음** |
| 저장소 | Cassandra·ES 등 | Cassandra·ES 등 | **오브젝트 스토리지만** |
| 비용 | 색인 DB 운영 | 색인 DB 운영(비쌈) | 스토리지만(저렴) |

- **Zipkin**은 분산추적의 원조 격이다(트위터, 2012).
- **Jaeger**는 우버가 만들어 CNCF로 간 프로젝트다.
  2024년 11월 나온 v2는 아예 OpenTelemetry Collector 위에 다시 지어졌다(구버전 v1은 2025년 말 지원 종료).
- **Tempo**는 비용에 초점을 맞췄다.
  별도의 색인 DB 없이 오브젝트 스토리지만으로 동작한다.

여기서 흔한 오해를 하나 바로잡자.
Tempo를 "색인이 전혀 없다"고 설명하는 글이 많은데, 정확하지 않다.
정확히는 "**외부 색인 DB(엘라스틱서치·Cassandra)가 필요 없고**, 오브젝트 스토리지 블록 안에 최소한의 색인(블룸 필터 등)만 둔다"이다.
대규모에서 저렴한 건 맞지만, 색인이 0인 건 아니다.

## 7.5 2026년 동향 — 코드 없는 계측(eBPF)

마지막으로 흥미로운 흐름 하나.
원래 계측은 코드를 건드려야 했다(라이브러리를 넣거나, 에이전트를 붙이거나).
요즘은 **eBPF**라는 리눅스 커널 기술로, **코드를 전혀 건드리지 않고** 커널 레벨에서 자동으로 추적하는 방식이 뜨고 있다.

Grafana의 Beyla가 OpenTelemetry에 기증돼 **OpenTelemetry eBPF Instrumentation(OBI)** 이 됐고, 2026년 KubeCon에서 베타가 공개됐다.
"애플리케이션을 안 고치고도 관측한다"는 방향이 점점 현실이 되고 있다.

## 7.6 정리

- OpenTelemetry는 **벤더 종속을 푸는** 계측 표준 — "한 번 계측, 어디로든".
- 구성: **API/SDK · OTLP(프로토콜) · semantic conventions(표준 이름) · Collector(중계)**.
- **Collector**(receiver→processor→exporter)를 거치면 앱 재배포 없이 파이프라인을 바꾼다.
- 추적 백엔드: Zipkin(원조)·Jaeger(우버/CNCF)·Tempo(오브젝트 스토리지·저렴).
- 2026년엔 **eBPF 기반 무계측**이 부상.

다음 장부터는 이렇게 모은 신호를 보고 다루는 쪽 — Grafana와 알림·SLO다.
