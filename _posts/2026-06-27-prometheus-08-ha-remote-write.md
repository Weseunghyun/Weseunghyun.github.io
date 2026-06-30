---
title: "Prometheus 완전 정복 8장 — 한 대로 안 될 때"
date: 2026-06-27 23:51:00 +0900
categories: ["Prometheus", "Prometheus 완전 정복"]
tags: ["prometheus", "운영"]
---
Prometheus 한 대는 가용성·장기보존·수평확장이 안 된다.
HA pair·federation·remote write·agent mode·sharding이 각각 **다른 문제**를 푼다.
헷갈리기 쉬우니 "무엇을 해결하나"로 구분하자.

## 8.1 HA pair — 가용성

같은 설정의 Prometheus **두 대**를 독립으로 돌린다.
한 대가 재시작·장애로 빠져도 다른 대가 데이터를 계속 쌓는다(4장의 WAL replay 사각지대를 메운다).

둘 다 같은 Alertmanager **클러스터**로 알림을 보내고, Alertmanager가 중복을 제거한다.
설계 철학은 "**누락보다 중복**" — 알림은 빠뜨리느니 두 번 가는 게 낫다.

한 가지 주의할 점.
두 Prometheus는 독립으로 긁어서 **데이터가 완전히 똑같지는 않다**(긁는 타이밍이 미세하게 다름).
그래서 쿼리 단에서 중복을 합치는 건 Thanos·Mimir 같은 **외부 계층**이 `replica` 라벨로 처리한다.

## 8.2 federation — 집계 메트릭 끌어오기

**federation**은 한 Prometheus가 다른 Prometheus의 `/federate`에서 **선택한 시계열의 현재 값**을 긁어오는 것이다.
보통 하위(데이터센터별) Prometheus가 상세를 갖고, 상위(글로벌) Prometheus가 **집계 메트릭만** 끌어와 통합한다.

주의 — federation은 **장기저장 대체가 아니다.**
"현재 값 스냅샷"만 끌어오는 것이지 전체 히스토리를 옮기는 게 아니다.
대량 시계열을 통째로 federate하면 상위가 터진다.

## 8.3 🔴 remote write — 장기저장에 흘려보내기

6장에서 본 장기저장으로 데이터를 보내는 길이 **remote write**다.
Prometheus가 WAL을 따라 데이터를 shard로 나눠 원격 엔드포인트(Mimir·Thanos·VictoriaMetrics)로 스트리밍한다.

운영에서 만지는 건 `queue_config`다 — shard 수, 큐 용량, 재시도 backoff.
원격이 느려지면 shard 큐가 차고 그게 WAL 읽기를 막는다(backpressure) → 로컬 메모리 압박.
그래서 원격 저장이 느려지면 Prometheus 본체도 영향을 받는다.

최신 **Remote Write 2.0**은 메타데이터·exemplar·native histogram을 함께 보내도록 개선됐다(아직 실험적 스펙).

## 8.4 agent mode — 수집 전용

엣지나 말단에서 "긁어서 중앙으로 보내기만" 하고 싶을 때가 있다.
**agent mode**(`--agent`)는 로컬 저장·쿼리·알림을 끄고 **scrape + remote write에만** 집중한다.
대규모로 메트릭을 중앙 장기저장으로 forward할 때 쓴다.

## 8.5 sharding — 부하 나누기

대상이 너무 많으면 Prometheus를 여러 대로 쪼갠다.
가장 단순한 건 job·팀·데이터센터별로 나누는 것(functional sharding)이다.
자동으로 나누려면 relabeling의 `hashmod`로 "대상 주소를 해시해서 N으로 나눈 나머지가 내 번호인 것만" 긁게 한다.

함정이 있다.
hashmod는 일관 해싱(consistent hashing)이 아니라서, **샤드 수 N을 바꾸면 거의 모든 대상이 다른 샤드로 재배치**된다.
샤드 수 조정은 신중해야 한다.

## 8.6 2026 — OpenTelemetry 수렴

운영 관점의 최신 흐름.

- **OTLP 수신**: `--web.enable-otlp-receiver`로 OpenTelemetry 메트릭을 직접 받는다.
- **3.x 마이그레이션 주의**: classic histogram의 `le` 값이 float으로 정규화돼서 `le="1"`을 직접 참조하던 알림·대시보드가 깨질 수 있다.
  scrape Content-Type도 엄격해졌고, remote write의 HTTP/2는 기본 꺼짐이다.

## 8.7 정리

- **HA pair**(가용성·외부 dedup) · **federation**(집계 스냅샷, 장기저장 대체 아님).
- **remote write**(장기저장에 스트리밍, backpressure 주의) · **agent mode**(수집 전용).
- **sharding**: hashmod는 consistent hashing 아님 → 샤드 수 변경 주의.
- 2026엔 OTLP 직접 수신, 3.x 마이그레이션 함정 주의.

다음 장에서 실제로 어디서 터지는지 모아 정리하고, 면접 체크리스트로 마무리한다.
