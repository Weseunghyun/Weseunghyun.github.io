---
title: "Prometheus 완전 정복 3장 — 긁어오기, 서비스 디스커버리, relabeling"
date: 2026-06-27 23:56:00 +0900
categories: ["Prometheus", "Prometheus 완전 정복"]
tags: ["prometheus", "핵심"]
---
1장에서 Prometheus는 대상을 긁어온다고 했다.
그럼 "무엇을 긁을지"는 어떻게 알고, 긁기 전에 어떻게 가공할까.
쿠버네티스처럼 파드가 수시로 뜨고 죽는 환경에서는 이게 핵심이다.

## 3.1 scrape 설정 기본

긁기 동작은 `scrape_config`로 정한다.
자주 보는 항목만 추리면 이렇다.

| 항목 | 의미 |
|---|---|
| `scrape_interval` | 긁는 주기(예: 15초) |
| `scrape_timeout` | 1회 긁기 타임아웃(interval보다 작아야) |
| `metrics_path` / `scheme` | 경로(기본 `/metrics`) / http·https |
| `sample_limit` | 한 번에 받을 최대 sample 수 — 초과하면 그 스크레이프 전체 실패 |
| `honor_labels` | 대상이 노출한 라벨과 자동 라벨이 충돌할 때 처리 |

특히 `sample_limit`은 카디널리티 방어의 1차 방어선이다 — 어떤 대상이 갑자기 수백만 시계열을 토하면 통째로 거부해 Prometheus를 지킨다(6장).

## 3.2 서비스 디스커버리 — 무엇을 긁을지 찾기

고정된 서버 목록을 손으로 적는 시대는 지났다.
Prometheus는 여러 소스에서 대상을 **동적으로 발견**한다.

| 방식 | 언제 |
|---|---|
| `static_configs` | 고정 호스트 목록 |
| `file_sd_configs` | 외부 스크립트가 파일로 대상 공급 |
| `kubernetes_sd_configs` | 쿠버네티스 API 기반(가장 흔함) |
| `consul_sd` · `ec2_sd` · `dns_sd` | Consul·EC2 태그·DNS |

쿠버네티스에서는 역할(role)로 무엇을 긁을지 고른다 — `node`(노드), `pod`(파드), `endpoints`(서비스 뒤 실제 파드, 앱 메트릭 표준), `service`(가용성 체크) 등.

## 3.3 🔴 relabeling — 긁기 전과 후의 가공

여기가 이 장에서 가장 중요하고 가장 헷갈리는 부분이다.
relabeling은 **두 단계**로 나뉜다.

| | `relabel_configs` | `metric_relabel_configs` |
|---|---|---|
| 시점 | **긁기 전** | **긁은 후, 저장 전** |
| 대상 | 어떤 대상을 긁을지 선택·주소·경로 가공 | 개별 메트릭을 가공·필터 |
| 쓰임 | "이 어노테이션 붙은 파드만 긁어라" | "이 위험한 라벨을 떼라" |

문법은 같다.
`source_labels`(읽을 라벨)를 보고, `regex`로 매칭해서, `action`(replace·keep·drop·labeldrop 등)을 적용한다.

쿠버네티스에서 흔한 패턴 두 개를 보자.

**긁을 대상 고르기** (relabel_configs, 긁기 전):
```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true          # prometheus.io/scrape=true 어노테이션 붙은 파드만
```

**위험한 라벨 떼기** (metric_relabel_configs, 저장 전):
```yaml
metric_relabel_configs:
  - regex: 'id|pod_uid'   # 폭발하는 라벨 이름 제거
    action: labeldrop
```

`__`로 시작하는 라벨(`__address__`·`__meta_*`)은 내부용이라 relabel이 끝나면 사라진다.
이걸 이용해 어노테이션 기반으로 주소·경로를 동적으로 바꾼다.

쿠버네티스에서 직접 이 YAML을 쓰는 일은 드물다 — 보통 Prometheus Operator의 ServiceMonitor/PodMonitor가 이 scrape 설정을 자동으로 만들어준다.

## 3.4 자동 라벨과 staleness

긁은 메트릭에는 Prometheus가 자동으로 `job`(설정한 이름)과 `instance`(대상 host:port) 라벨을 붙인다.

그리고 **staleness(낡음)** 처리가 있다.
어떤 대상이 사라지거나 긁기에 실패하면 Prometheus는 그 시계열에 "낡음 표시"를 붙여서, 그 시점 이후로는 값을 반환하지 않는다.
"데이터가 0"이 아니라 "데이터 없음"으로 처리하는 것이다.
그래서 대상이 살아있는지는 `up`이라는 별도 메트릭으로 본다(`up == 1`이면 정상).

## 3.5 Exporter와 Pushgateway

- **Exporter**: 직접 계측이 안 되는 시스템(하드웨어·DB)을 `/metrics`로 변환해주는 어댑터다.
  node_exporter(호스트)·blackbox_exporter(외부 프로빙)가 대표다.
  내 코드라면 client library로 직접 계측하는 게 정석이다.
- **Pushgateway**: 1장에서 말한 단명 배치용 다리다.
  다만 보통은 안티패턴인데, push된 시계열은 **자동 낡음 처리가 안 돼서** 잡이 죽어도 마지막 값이 영원히 남는다.
  머신 종속 배치라면 node_exporter의 textfile 방식이 더 낫다.

## 3.6 정리

- `scrape_config`로 긁기 정의, `sample_limit`이 카디널리티 1차 방어.
- 대상은 **서비스 디스커버리**로 동적 발견(쿠버네티스 role).
- **relabeling 두 단계**: 긁기 전(`relabel_configs`, 대상 선택)·저장 전(`metric_relabel_configs`, 메트릭 가공).
- 대상이 사라지면 **staleness** 처리, 가용성은 `up`으로.

다음 장에서는 긁어온 숫자가 디스크에 어떻게 저장되는지 — TSDB의 내부를 연다.
