---
title: "Grafana 완전 정복 6장 — Observability as Code + 마스터 체크"
date: 2026-06-27 23:53:00 +0900
categories: ["Grafana", "Grafana 완전 정복"]
tags: ["grafana", "마무리"]
---
## 6.1 대시보드를 코드로

대시보드를 UI에서 손으로 만들고 끝내면, 누가 언제 무엇을 바꿨는지 추적이 안 된다.
대시보드·알림이 늘수록 이걸 **코드처럼** 다뤄야 한다 — Observability as Code.

| 방법 | 내용 |
|---|---|
| **File provisioning** | YAML로 데이터소스·대시보드를 파일에서 선언·주입 |
| **Git Sync** | 대시보드를 Git에 저장, UI에서 브랜치·PR·머지 (Grafana 13 GA) |
| **Foundation SDK** | Go·TS·Python 등으로 대시보드를 타입 안전 코드로 정의 |
| **Terraform** | HCL로 대시보드·알림 관리 |

런타임 기반은 **Scenes**라는 라이브러리이고(2024 GA), Grafana 13은 Dynamic Dashboards·Grafana Assistant(AI) 같은 기능을 더했다.

## 6.2 개념 체크리스트

- [ ] Grafana가 **무엇을 저장하나**(메타만, 시계열은 데이터소스에)
- [ ] **query proxy**가 왜 필요한가(인증·자격증명·통일 인터페이스)
- [ ] transformation은 어디서 실행되나(프론트, 순서 중요)
- [ ] correlation(exemplar·trace→logs)이 단일 창을 어떻게 만드나
- [ ] Unified Alerting: rule·상태·notification policy
- [ ] org/RBAC/폴더 경계, HA(공유 DB·gossip 9094)
- [ ] Observability as Code(Git Sync·Foundation SDK)

## 6.3 면접 Q&A

**Q.
Grafana는 데이터를 저장하나요?**
아니요.
대시보드·유저 같은 메타데이터만 자체 DB에 저장합니다.
시계열·로그·트레이스는 외부 데이터소스에 있고, Grafana는 query proxy로 쿼리를 위임해 보여줍니다.
그래서 어떤 백엔드든 중립적으로 붙일 수 있습니다.

**Q.
한 화면에서 메트릭→트레이스→로그로 어떻게 넘어가나요?**
correlation 기능입니다.
메트릭의 exemplar로 trace에 점프하고, trace span에서 trace-to-logs로 Loki 로그에 점프합니다.
백엔드가 exemplar를 노출하고 로그에 trace_id를 박아두면 Grafana가 그 링크를 만들어줍니다.

**Q.
Grafana 알림에서 NoData/Error를 조심해야 하는 이유는?**
기본 설정은 데이터가 없으면 NoData, 쿼리가 실패하면 Error로 처리하는데, 데이터소스 일시 장애에 규칙이 의도치 않게 발화할 수 있습니다.
상황에 따라 "마지막 상태 유지"를 고려합니다.

**Q.
Grafana를 여러 대로 운영하려면?**
여러 인스턴스가 공유 DB(MySQL/Postgres)를 바라보게 하고, 알림은 gossip(9094)으로 상태를 동기화합니다.
SQLite는 단일 인스턴스용이라 HA엔 안 맞습니다.

## 6.4 마치며

Grafana의 모든 것은 한 결정에서 나온다.
**저장하지 않고 쿼리만 위임한다.**
그래서 어떤 백엔드든 중립적으로 붙고(단일 창), 신호 사이를 correlation으로 잇고, 자신은 시각화·알림·권한에 집중한다.

이걸로 관측가능성 도구 다섯(Prometheus·OpenTelemetry·Loki·Tempo·Grafana)을 한 바퀴 돌았다.
메트릭을 모으고(Prometheus), 표준으로 계측하고(OpenTelemetry), 로그·트레이스를 싸게 저장하고(Loki·Tempo), 한 화면에서 본다(Grafana).
이 다섯이 맞물리면 새벽의 "어디가 느린지 모르겠다"가 "메트릭→트레이스→로그 3분"이 된다.
