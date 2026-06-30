---
title: "Tempo 완전 정복 4장 — TraceQL"
date: 2026-06-27 23:55:00 +0900
categories: ["Tempo", "Tempo 완전 정복"]
tags: ["tempo", "핵심"]
---
TraceQL은 트레이스를 검색하는 언어다.
PromQL·LogQL과 결이 비슷하지만, **trace 계층(부모-자식)을 따지는 연산자**가 고유하다.

## 4.1 spanset selector `{ }`

기본은 중괄호 안에 조건을 적는 것이다.
조건이 참인 **span 집합(spanset)** 이 결과다.

- intrinsic(내장 필드, `:` 스코프): `{ span:duration > 1s }`, `{ span:status = error }`, `{ trace:duration > 5s }`
- 커스텀 속성(`.` 스코프): `{ span.http.method = "GET" }`, `{ resource.service.name = "frontend" }`

🔴 스코프(`span.`·`resource.`)를 **꼭 붙이는 게 좋다.**
생략하면 여러 곳을 다 확인해서 느려진다.

## 4.2 연산자와 집계

- 비교 `= != > >= < <=`, 정규식 `=~`, 논리 `&&` `||`.
- 파이프로 집계: `{ span.http.status_code = 200 } | count() > 3`, `| avg(span:duration)`.
- 그룹핑·선택: `| by(resource.service.name)`, `| select(...)`.

## 4.3 🔴 structural operators — 계층 따지기

이게 TraceQL의 고유함이다.
trace의 부모-자식 관계로 span을 거른다.

| 연산자 | 의미 |
|---|---|
| `>>` / `<<` | descendant(후손) / ancestor(조상) |
| `>` / `<` | child(직속 자식) / parent |
| `~` | sibling(형제) |

예: `{ resource.service.name="frontend" } >> { span:status = error }`
→ "frontend 하위 어딘가에서 에러난 span"을 찾는다.
이런 "A 밑에서 B가 일어난" 질의는 메트릭·로그로는 못 하는, 트레이스만의 검색이다.

## 4.4 흔한 쿼리 예시

```
{ resource.service.name = "frontend" && span:status = error }
{ span.http.status_code >= 500 }
{ trace:duration > 5s }
{ resource.deployment.environment = "production" && trace:duration > 2s } | count() > 30
```

## 4.5 정리

- **spanset selector `{}`** + intrinsic(`span:`)·속성(`span.`) — **스코프를 붙여라**.
- 집계는 `| count()`·`| avg()`·`| by(...)`.
- **structural operators**(`>>`·`>`·`~`)로 trace 계층을 따지는 게 TraceQL 고유.

다음 장에서는 트레이스에서 메트릭을 뽑는 두 방법을 본다.
