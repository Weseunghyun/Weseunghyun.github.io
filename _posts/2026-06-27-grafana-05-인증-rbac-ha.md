---
title: "Grafana 완전 정복 5장 — 인증·RBAC·HA"
date: 2026-06-27 23:54:00 +0900
categories: ["Grafana", "Grafana 완전 정복"]
tags: ["grafana", "운영"]
---
여러 팀이 한 Grafana를 쓰면 "누가 무엇을 볼·고칠 수 있나"와 "한 대 죽어도 되나"가 문제가 된다.
멀티테넌시·권한·고가용성을 본다.

## 5.1 org · team · user

- **organization(org)**: 인스턴스 안의 격리 단위 — **멀티테넌시의 기본**.
  모든 유저는 최소 1개 org에 속한다.
- **Server admin ≠ Org admin**: server admin은 서버 전역(org·유저·라이선스), org admin은 그 org 자원만.
- **team**: org 안의 권한 그룹.

## 5.2 권한 — basic role + RBAC

기본 역할은 셋이다.

| 역할 | 권한 |
|---|---|
| **Admin** | org 전 권한 |
| **Editor** | 대시보드·폴더 편집(유저·데이터소스 관리 불가) |
| **Viewer** | 조회·쿼리 |

더 세밀하게는 **RBAC**(fine-grained, Enterprise/Cloud 전용)가 있다 — action+scope 조합으로 커스텀 역할을 만들고 유저·팀·서비스 계정에 부여한다.
**폴더가 주요 권한 경계**라, 폴더 단위로 누가 어떤 대시보드를 보는지 통제한다.

## 5.3 SSO

기업 환경에서는 SSO로 연동한다.

- **Generic OAuth/OIDC·SAML(Enterprise)·LDAP·Microsoft Entra ID** 등.
- **Team sync**(IdP 그룹 ↔ Grafana 팀), role 자동 매핑.
- Grafana 10+부터 SSO를 파일·재시작 없이 UI에서 설정한다.

## 5.4 🔴 HA

Grafana를 여러 대로 띄울 때.

- **여러 인스턴스 + 공유 DB**(MySQL/Postgres)가 기본 구성이다.
  SQLite는 단일 인스턴스용이라 HA엔 안 맞는다.
- 알림은 평가(Scheduler)와 전달(Alertmanager)을 나누고, 기본적으로 **모든 인스턴스가 모든 규칙을 평가**한다(이중화).
- 인스턴스끼리 **gossip 프로토콜(port 9094)** 로 알림·silence 상태를 주고받는다.
  dedup은 best-effort라 가끔 중복 알림이 갈 수 있다(가용성 우선).

## 5.5 2026 동향

- **Grafana IRM**(구 OnCall + Incident 통합): 2025-03 클라우드 롤아웃.
  🔴 **Grafana OnCall OSS는 2026-03 아카이브** — OSS 사용자는 마이그레이션이 필요하다.
- Grafana 13은 RBAC enforcement를 강화했다(일부 breaking).

## 5.6 정리

- 멀티테넌시는 **org**, 권한은 basic role(Viewer/Editor/Admin) + **RBAC**(Enterprise), 폴더가 경계.
- SSO(OAuth/SAML/LDAP) + team sync.
- HA는 **여러 인스턴스 + 공유 DB**, 알림은 gossip(9094)로 동기화.
- **OnCall OSS 아카이브**(2026-03) 주의.

다음 장에서는 대시보드를 코드로 관리하는 법과 면접 정리다.
