---
title: "Prometheus 완전 정복 4장 — 저장의 내부, TSDB"
date: 2026-06-27 23:55:00 +0900
categories: ["Prometheus", "Prometheus 완전 정복"]
tags: ["prometheus", "핵심"]
---
긁어온 숫자는 Prometheus 안의 시계열 데이터베이스(TSDB)에 저장된다.
이 내부 구조를 알아야 6장의 "왜 죽는가"가 이해된다.
핵심 한 줄을 먼저 박아두자.

> 최근 데이터는 **메모리에 산다.**
> 그래서 시계열 개수가 곧 메모리 사용량이다.

## 4.1 2계층 — 메모리와 디스크

TSDB는 메모리의 **head block**과 디스크의 **block** 두 계층으로 나뉜다.

![](/assets/img/posts/prometheus-04-tsdb-내부/prometheus-tsdb-lifecycle.svg)

새 샘플이 들어오면 이런 길을 간다.

1. **WAL(write-ahead log)**: 메모리에 반영하기 **전에** 디스크의 로그에 먼저 적는다.
   갑자기 죽어도 복구할 수 있게.
2. **head block(메모리)**: 최근 데이터가 메모리에 쌓인다.
3. **mmap chunks**: head에서 다 찬(더 안 쓰는) chunk는 디스크로 메모리 매핑해 메모리 압력을 던다.
4. **block(디스크)**: 일정 시간(기본 2시간)이 지나면 immutable한 block으로 flush된다.
5. **compaction**: 작은 block들을 큰 block으로 병합하고, 보관 기간이 지난 건 삭제한다.

## 4.2 WAL과 재시작

WAL은 안전장치다.
모든 샘플이 메모리에 반영되기 전에 WAL에 먼저 기록되니, 프로세스가 갑자기 죽어도 재시작 때 WAL을 다시 읽어(replay) 메모리 상태를 복구한다.

여기에 함정이 하나 있다.
시계열이 많아 WAL이 크면 **재시작이 느려진다** — 그 큰 WAL을 다 읽어야 하기 때문이다.
운영하다 "Prometheus 재시작했더니 몇 분간 데이터가 안 보였다"는 게 이것이다.
그 동안의 사각지대는 보통 같은 설정의 Prometheus 두 대(HA pair, 8장)로 메운다.

## 4.3 block과 compaction

2시간마다 만들어지는 block은 **immutable**(한번 만들면 안 바뀜)이다.
디렉토리 안에 chunks(압축된 데이터)·index(어디에 무엇이 있는지)·meta로 구성된다.

compaction은 두 가지를 한다.
작은 block을 큰 block으로 **병합**해(쿼리할 때 뒤질 block 수를 줄임) 성능을 올리고, 보관 기간(retention)이 지난 block을 통째로 삭제한다.

보관 기간 기본값은 15일이고, 시간 기준(`retention.time`)이나 크기 기준(`retention.size`)으로 정한다.
삭제 단위는 **block 디렉토리 통째**다.

## 4.4 index — 어떻게 빨리 찾나

`http_requests_total{code="500"}` 같은 쿼리가 들어오면 Prometheus는 어떻게 해당 시계열을 빨리 찾을까.
**inverted index(역색인)** 다 — "code=500이라는 라벨을 가진 시계열들" 목록을 미리 정리해둔다.
라벨 문자열은 정수로 바꿔(symbol table) 저장해 공간과 속도를 아낀다.

## 4.5 압축 — 숫자가 작아지는 마법

시계열은 "비슷한 숫자가 규칙적으로 쌓이는" 데이터다.
이 특성을 이용해 아주 잘 압축한다.

- timestamp는 간격의 간격(delta-of-delta)만 저장한다 — 15초마다 찍으면 그 "15초"가 거의 안 변하니 거의 0에 가까운 값만 남는다.
- 값은 직전 값과 XOR해 변한 비트만 저장한다.

이 방식(Gorilla 인코딩) 덕에 메트릭이 그렇게 싸다.

한 가지 흔한 오해를 짚자.
"chunk 하나에 샘플 120개"라는 말이 도는데, 정확히는 **고정 개수가 아니라 바이트 한계(약 1KB)** 기준이다.
운영상 흔히 120개쯤에서 잘릴 뿐, 하드코딩된 상수가 아니다.

## 4.6 정리

- TSDB는 **메모리(head) + 디스크(block)** 2계층, **WAL**로 안전.
- 최근 데이터가 메모리에 → **시계열 수 = 메모리** (6장 카디널리티의 뿌리).
- block은 2시간 단위 immutable, **compaction**으로 병합·삭제.
- **역색인**으로 빠르게 찾고, **delta-of-delta + XOR**로 압축.

다음 장에서는 이 저장된 데이터에 질문하는 언어, PromQL이다.
