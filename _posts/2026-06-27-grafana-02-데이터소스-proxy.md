---
title: "Grafana 완전 정복 2장 — 데이터소스와 query proxy"
date: 2026-06-27 23:57:00 +0900
categories: ["Grafana", "Grafana 완전 정복"]
tags: ["grafana", "핵심"]
---
Grafana가 "저장 안 하고 쿼리만 한다"는 게 1장의 핵심이었다.
그 쿼리 위임이 어떻게 동작하는지 본다.

## 2.1 query proxy — 백엔드가 쿼리를 위임받는다

![](/assets/img/posts/grafana-02-데이터소스-proxy/grafana-query-proxy.svg)

대시보드 패널이 쿼리를 던지면, Grafana **백엔드**가 그걸 받아 해당 데이터소스로 전달한다(프록시).
응답은 data frame이라는 공통 형태로 돌아와 패널이 그린다.

**왜 백엔드를 거치나(proxy 모드).**

- 인증·인가를 백엔드 도달 전에 강제한다.
- 데이터소스 자격증명을 서버에서 관리한다(브라우저에 노출 안 함).
- 이종 소스에 **통일된 쿼리 인터페이스**를 준다.

(브라우저가 데이터소스에 직접 붙는 direct 모드도 있지만, CORS·보안 노출 때문에 권장되지 않는다.)

## 2.2 데이터소스 플러그인

데이터소스마다 전용 플러그인이 있고, 각자의 쿼리 에디터를 제공한다.

- Prometheus → PromQL 에디터
- Loki → LogQL 에디터
- Tempo → TraceQL 에디터

빌트인만 170개가 넘고, 커뮤니티·마켓플레이스 플러그인도 있다.

## 2.3 Mixed data source

한 패널에서 **여러 데이터소스를 동시에** 쿼리할 수도 있다(`-- Mixed --`).
예를 들어 Prometheus 메트릭과 SQL 비즈니스 데이터를 한 그래프에 겹쳐 본다.
Grafana 13부터는 쿼리별로 데이터소스를 고르는 picker가 기본이라 더 직관적이다.

## 2.4 정리

- Grafana 백엔드가 **query proxy**로 데이터소스에 쿼리를 위임한다 — 인증·자격증명·통일 인터페이스를 위해.
- 데이터소스마다 전용 플러그인·쿼리 에디터(PromQL·LogQL·TraceQL).
- **Mixed**로 한 패널에 여러 소스를 겹쳐 볼 수 있다.

다음 장에서는 대시보드·변수·변환, 그리고 신호 연결(correlation)을 본다.
