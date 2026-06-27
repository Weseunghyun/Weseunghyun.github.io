---
title: "관측가능성 완전 정복 11장 — 직접 해보기"
date: 2026-06-25 23:48:00 +0900
categories: ["Observability", "관측가능성 완전 정복"]
tags: ["observability", "실전"]
---
이론은 충분히 봤다.
이 장에서는 실제로 서비스를 계측하고, 로컬에서 스택을 띄워본다.
큰 그림은 단순하다.

> 앱에 OpenTelemetry를 붙여 신호를 만들고 → 보통 Collector를 거쳐 → 백엔드(Tempo·Prometheus·Loki)로 보낸다.

## 11.1 계측 — 자동이냐 수동이냐

앱에 OpenTelemetry를 붙이는 방법은 둘이다.

**자동 계측(zero-code).**
코드를 안 고치고 붙이는 방식이다.
예를 들어 자바(Spring Boot)라면, 에이전트 jar 하나를 실행 옵션으로 끼우면 된다.

```bash
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=my-api \
     -Dotel.exporter.otlp.endpoint=http://collector:4318 \
     -jar app.jar
```

이 에이전트가 바이트코드를 조작해, 들어오는 요청·나가는 HTTP 호출·DB 호출을 알아서 추적한다.
.NET·Go·Node.js·PHP·Python도 비슷한 자동 계측을 지원한다.

**수동 계측.**
자동이 못 잡는 비즈니스 로직 구간에 직접 span이나 메트릭을 추가하는 방식이다.
보통 **자동 + 수동을 함께** 쓴다.
자동으로 큰 틀을 잡고, 중요한 도메인 구간만 수동으로 보강한다.

## 11.2 Collector를 어디에 둘 것인가

7장에서 본 Collector를 어떻게 배치하느냐는 세 가지 패턴이 있다.

| 패턴 | 설명 |
|---|---|
| **Collector 없음** | 앱이 백엔드로 직접 전송. 간단하지만 바꾸려면 앱 재배포 |
| **Agent** | 노드/파드 옆에 Collector. 계측과 전송을 분리 |
| **Gateway** | 중앙에 Collector 한 곳. 샘플링·필터·여러 백엔드 분배 |

흔한 조합은 **노드마다 Agent → 중앙 Gateway → 백엔드**다.
tail 샘플링 같은 무거운 처리는 중앙 Gateway에서 한다(10장의 "같은 요청은 같은 수집기로" 문제 때문에 라우팅이 필요하다).

## 11.3 꼭 알아야 할 설정값

Collector 설정에서 빠지면 안 되는 것들이다.

- **OTLP endpoint**: 데이터를 받는 주소.
  gRPC는 4317, HTTP는 4318 포트.
- **service.name**: 이 서비스의 이름.
  **이게 없으면 백엔드에서 누가 보낸 건지 식별이 안 된다.** 사실상 필수.
- **batch processor**: 신호를 묶어 보내 네트워크 효율을 높인다.
  거의 항상 넣는다.
- **memory_limiter processor**: Collector가 메모리 부족으로 죽는 걸 막는다.
  한도를 넘으면 데이터를 버려서라도 살아남는다.
  **파이프라인 맨 앞**에 둬야 한다.

Collector 설정 골격은 이렇게 생겼다.

```yaml
receivers:
  otlp:
    protocols: { grpc: {}, http: {} }
processors:
  memory_limiter: { check_interval: 1s, limit_mib: 1024 }   # 맨 앞
  batch: {}
exporters:
  otlp/tempo: { endpoint: tempo:4317, tls: { insecure: true } }
  prometheusremotewrite: { endpoint: http://mimir:9009/api/v1/push }
service:
  pipelines:
    traces:  { receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlp/tempo] }
    metrics: { receivers: [otlp], processors: [memory_limiter, batch], exporters: [prometheusremotewrite] }
```

## 11.4 로컬에서 한 번에 띄우기

직접 굴려보는 게 가장 빠른 학습이다.
Grafana가 학습용으로 공개한 저장소 두 개를 쓰면 된다.

| 저장소 | 무엇 |
|---|---|
| **`grafana/intro-to-mltp`** | Collector + Prometheus + Tempo + Loki + Grafana + **샘플 마이크로서비스**까지 docker-compose로 한 번에 |
| **`grafana/docker-otel-lgtm`** | 위 백엔드들을 **단일 컨테이너**로 묶음(샘플 앱은 없음). 내 앱을 이미 계측했을 때 |

처음 배운다면 `intro-to-mltp`가 좋다.
샘플 앱이 트래픽을 만들어주니, Grafana에 들어가 메트릭→트레이스→로그로 점프하는 2장의 그 흐름을 직접 눌러볼 수 있다.

```bash
git clone https://github.com/grafana/intro-to-mltp
cd intro-to-mltp
docker compose up -d
# Grafana: http://localhost:3000
```

실제로 띄워보면 Grafana·Mimir·Loki·Tempo·Pyroscope·Alloy에 샘플 앱과 부하 생성기(k6)까지 십여 개 컨테이너가 한 번에 뜬다.
잠시 두면 세 신호가 실제로 흘러든다.
메트릭 쪽에 `up` 같은 기본 지표가 쌓이고, 로그에 서비스 라벨이 잡히고, 트레이스에는 샘플 앱의 요청들이(가끔 1~2초씩 느린 요청까지) 들어온다.
Grafana에 접속해 메트릭 그래프에서 트레이스로, 다시 로그로 점프하는 2장의 그 흐름을 직접 눌러볼 수 있다.
(버전·구성은 저장소 README 기준으로 바뀔 수 있으니 확인하자.)

## 11.5 자주 쓰는 패턴

마지막으로 실무에서 반복되는 패턴 셋.

- **RED 메트릭**: 서비스마다 요청수·에러·지연을 노출한다.
  Tempo는 트레이스에서 이 RED 메트릭을 자동으로 만들어주기도 한다.
- **로그에 trace_id 박기**: 로그를 JSON으로 남기면서 현재 `trace_id`를 필드로 넣는다.
  그래야 8장의 로그↔트레이스 점프가 된다.
- **exemplar 연결**: 히스토그램에 trace_id를 붙여, 메트릭 그래프의 점에서 트레이스로 점프한다.

세 가지를 다 갖추면 2장에서 말한 **메트릭→트레이스→로그 왕복**이 완성된다.

## 11.6 정리

- 계측은 **자동(에이전트) + 수동** 혼합.
- Collector는 **Agent / Gateway** 패턴, `memory_limiter`는 맨 앞.
- **service.name 필수**, OTLP 포트는 4317/4318.
- 로컬은 **`grafana/intro-to-mltp`** 로 한 번에 띄워 직접 눌러본다.
- 패턴: RED 메트릭 · 로그에 trace_id · exemplar.

다음 장은 실제 회사들이 이걸 어떻게 쓰는지 — 대기업 레퍼런스다.
