---
title: "OpenTelemetry 완전 정복 7장 — 직접 해보기 + 마스터 체크"
date: 2026-06-27 23:52:00 +0900
categories: ["OpenTelemetry", "OpenTelemetry 완전 정복"]
tags: ["opentelemetry", "마무리"]
---
마지막 장이다.
실제로 계측하는 큰 그림을 보고, 면접에서 설명할 수준으로 정리한다.

## 7.1 직접 계측 — 큰 그림

앱에 OTel을 붙이는 흐름은 단순하다.

> 앱을 계측 → 보통 Collector를 거쳐 → 백엔드로.

자동 계측 실행 예는 이렇다.

```bash
# Java
java -javaagent:opentelemetry-javaagent.jar \
     -Dotel.service.name=my-api \
     -Dotel.exporter.otlp.endpoint=http://collector:4318 \
     -jar app.jar

# Python
OTEL_SERVICE_NAME=my-api OTEL_EXPORTER_OTLP_ENDPOINT=http://collector:4318 \
  opentelemetry-instrument python app.py
```

빠뜨리면 안 되는 환경변수는 `OTEL_SERVICE_NAME`(필수)·`OTEL_EXPORTER_OTLP_ENDPOINT`·`OTEL_TRACES_SAMPLER`다.

## 7.2 로컬에서 체험

관측가능성 시리즈에서 띄워본 `grafana/intro-to-mltp`가 그대로 쓸 수 있다.
Collector(Alloy) + Prometheus/Mimir·Loki·Tempo·Grafana + 샘플 앱이 docker-compose로 한 번에 뜨고, OTel 데이터가 흐르는 걸 직접 눌러볼 수 있다.

```bash
git clone https://github.com/grafana/intro-to-mltp
cd intro-to-mltp && docker compose up -d   # Grafana: localhost:3000
```

## 7.3 개념 체크리스트

- [ ] OpenTelemetry가 푸는 문제 (벤더 종속 탈출, "한 번 계측 어디로든")
- [ ] **API/SDK를 왜 나눴나** (라이브러리는 API만, no-op default)
- [ ] 신호별 구조 (Provider→...→Processor→Exporter), BatchSpanProcessor 5초
- [ ] 자동 계측의 언어별 메커니즘 (Java 바이트코드·Python monkeypatch·Go eBPF)
- [ ] Collector를 왜 거치나, **memory_limiter 맨 앞**, agent vs gateway
- [ ] OTLP (gRPC 4317/HTTP 4318, Resource→Scope→Span)
- [ ] **semantic conventions**가 왜 신호 연결의 전제인가
- [ ] context가 큐·async에서 끊기는 이유

## 7.4 면접 Q&A

**Q.
OpenTelemetry가 왜 중요한가요?**
계측을 표준화해 벤더 종속에서 벗어나게 합니다.
한 번 계측하면 앱을 안 고치고도 백엔드를 바꿀 수 있고, 2026년 5월 CNCF를 졸업해 사실상의 표준이 됐습니다.

**Q.
API와 SDK를 왜 분리했나요?**
라이브러리는 API에만 의존하게 해서, 어떤 OTel 구현이 깔릴지 몰라도 되게 합니다.
SDK가 없으면 API 호출이 no-op으로 떨어져 오버헤드가 거의 없습니다 — 그래서 라이브러리가 OTel 안 쓰는 앱에서도 멀쩡히 돕니다.

**Q.
Collector를 왜 거치나요?**
앱 재배포 없이 보내는 대상을 바꾸고, 배치·압축·재시도로 효율을 높이고, 샘플링·민감정보 제거를 중앙에서 합니다.
memory_limiter를 맨 앞에 둬 OOM을 막습니다.

**Q.
trace가 중간에 끊기는 흔한 이유는?**
context가 보통 스레드에 묶여 있어, 비동기·메시지 큐 경계로 넘어가면 자동으로 따라가지 않습니다.
큐에 넣을 때 헤더에 context를 싣고 꺼낼 때 복원해야 합니다.

**Q.
semantic conventions가 왜 필요한가요?**
모두가 `service.name`·`http.request.method` 같은 표준 이름을 쓰기 때문에 메트릭·로그·트레이스가 서로 묶이고 백엔드가 데이터를 이해합니다.
이름이 제각각이면 신호 연결이 안 됩니다.

## 7.5 마치며

OpenTelemetry의 핵심은 한 문장이다.
**계측은 한 번, 표준으로.**
그 표준이 전송에선 OTLP, 이름에선 semantic conventions이고, 중간 허브가 Collector다.

이 구조를 알면 어떤 백엔드(Prometheus·Loki·Tempo·Datadog)를 쓰든 계측 코드는 그대로 두고 갈아끼울 수 있다 — 그게 OTel이 표준이 된 이유다.
