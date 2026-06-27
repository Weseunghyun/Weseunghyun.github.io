---
title: "관측가능성 완전 정복 12장 — 실제 회사들은 어떻게 쓰나"
date: 2026-06-25 23:47:00 +0900
categories: ["Observability", "관측가능성 완전 정복"]
tags: ["observability", "레퍼런스"]
---
이론과 도구를 봤으니, 실제 회사들이 어떻게 쓰는지 본다.
여기 적은 사례는 모두 **그 회사가 직접 공개한 자료**(엔지니어링 블로그·공식 발표)를 기준으로 했다.
개인 블로그 요약이나 "들었다"는 사례는 넣지 않았고, 못 찾은 건 못 찾았다고 적었다.
수치에는 시점을 함께 적는다.

## 12.1 공통 패턴 먼저

여러 사례를 관통하는 흐름이 있다.

- **ELK가 비싸서 Loki·컬럼형 DB로** 옮기는 흐름
- Prometheus 한 대의 한계를 **Thanos·Mimir·VictoriaMetrics로** 확장
- 계측을 **OpenTelemetry로** 표준화
- 로그·트레이스 백엔드로 **ClickHouse**가 부상

이걸 염두에 두고 사례를 보면 패턴이 반복되는 게 보인다.

## 12.2 글로벌

| 회사 | 무엇을 왜 | 출처(시점) |
|---|---|---|
| **Netflix** | 메트릭 폭증(2014년 12억 metric)으로 자체 시계열 플랫폼 **Atlas** 구축 | Netflix TechBlog (2014-12) |
| **Uber** | 4천여 마이크로서비스 → 메트릭 플랫폼 **M3** + 분산추적 **Jaeger**(우버가 만들어 CNCF로) | Uber Engineering Blog |
| **Cloudflare** | 초당 약 100만 로그 라인 → ELK + **ClickHouse** 병행(문서 600B→행 60B) | Cloudflare Blog (2024-01) |
| **Grafana Labs** | LGTM 자체 검증 → **Mimir로 10억 active series** 부하테스트 | Grafana Blog (2022-04) |
| **AWS** | Amazon Managed Prometheus는 **Cortex 기반**, 20개 리전 500억 시계열 | KubeCon EU 2025 (2025-04) |
| **Spotify** | 자체 TSDB 한계 → **VictoriaMetrics**(초당 7,800만 datapoint) | VictoriaMetrics 케이스 (2025-12) |

특히 데이터독의 2023년 장애 사후분석은 10장에서 다뤘듯 "관측 시스템이 대상과 함께 죽는" 함정의 교과서다.

## 12.3 국내 (한국)

국내 회사들도 1차 자료가 꽤 있다.

**우아한형제들(배달의민족)** — ELK → LGTM 전환.
ELK의 색인 부하와 비용, 안정성 문제로 Grafana LGTM 스택(Loki·Tempo·Mimir)으로 옮긴 이야기다.
레이블만 색인하는 Loki로 바꿔 압축률을 높였고, 로그 수집량은 1.6배로 늘었는데 **비용은 약 23% 줄었다**고 공개했다(자사 기술블로그, 2023-11).
5장에서 본 "Loki가 왜 싼가"의 실제 사례다.

**토스증권** — Kafka 데이터 리니지.
파드 약 1만 개, 토픽 약 1천 개 규모의 Kafka 환경에서, 어떤 서비스가 어떤 토픽을 쓰는지 코드 수정 없이 추적하는 시스템을 만들었다.
Kafka의 메타데이터 API 로그를 **ClickHouse**에 적재하고, 컨슈머 지연 정보와 조인해 리니지를 시각화한다(자사 블로그).

**LINE / LY** — Grafana Alloy와 SLI/SLO.
프론트엔드부터 백엔드까지 끝까지 추적하기 위해 **Grafana Alloy**(OpenTelemetry Collector)와 Faro를 써서 Prometheus·Loki·Tempo로 보낸다(2024-09).
또 약 160개 서비스가 쓰는 플랫폼에 SLI/SLO를 도입한 사례도 공개했는데, **하루 약 350TB 로그** 규모다(2025-04).

**SK(DevOcean)** — Prometheus 3.0.
쿠버네티스 플랫폼 팀이 Prometheus 3.0 업그레이드(OpenTelemetry 상호운용 강화)를 다룬 글을 공개했다(2024-12).

**당근** — EKS 위의 Prometheus + Loki.
AWS EKS 기반으로 Prometheus와 Loki를 운영하고, 감사 로그를 EventBridge로 받아 모든 kubectl 동작을 Slack으로 보내는 식의 운영을 공개했다(2024~2026).

## 12.4 🔴 못 찾은 것은 못 찾았다고

정직하게 적자.
다음은 이번 조사에서 **회사의 1차 공개 자료를 확인하지 못한** 경우다.

- **NAVER**: 자사 관측가능성 구축기를 공개 1차 출처로 특정하지 못했다(검색 결과는 대부분 개인 블로그였다).
  APM 도구 Pinpoint가 NAVER 출신 오픈소스라는 건 알려져 있지만, 자사 적용기 1차 자료는 확보하지 못했다.
- **쿠팡**: 공식 엔지니어링 블로그에서 관측가능성 주제의 1차 글을 찾지 못했다.
- **카카오**: 분산추적 관련 글의 제목·날짜는 확인했으나 본문을 검증하지 못했다.

이건 "그 회사가 안 한다"는 뜻이 아니라, **이번 검색에서 1차 자료를 못 찾았다**는 뜻이다.
사례를 지어내기보다 한계를 적는 게 맞다(이 시리즈 전체의 원칙이다).

## 12.5 정리

- 공통 흐름: **ELK→Loki/ClickHouse(비용), Prometheus 확장(Thanos·Mimir·VM), 계측 OTel 표준화**.
- 글로벌: Netflix(Atlas)·Uber(M3·Jaeger)·Cloudflare·Grafana(Mimir 10억)·AWS(Cortex)·Spotify(VM).
- 국내: 우아한(ELK→LGTM, 비용 23%↓)·토스증권(ClickHouse 리니지)·LINE(Alloy·SLO 350TB)·SK·당근.
- 못 찾은 사례는 **못 찾았다고** 적는다.

마지막 장에서 전체를 체크리스트와 면접 Q&A로 정리한다.
