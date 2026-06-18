---
title: "Docker 완전 정복 5장 — Dockerfile 잘 쓰기 (멀티스테이지·캐시·비-root)"
date: 2026-06-17 23:54:00 +0900
categories: ["Docker", "Docker 완전 정복"]
tags: ["docker", "실전"]
---
4장에서 이미지가 레이어의 층층이라는 원리를 배웠다.
이 장은 그 원리를 실전 규칙으로 모은다.

좋은 Dockerfile은 세 가지를 동시에 노린다.
**빌드가 빠르고(캐시), 이미지가 작고, 운영에서 안전한** 것.
하나씩 보자.

## 5.1 좋은 Dockerfile의 전체 모습

먼저 잘 짜인 Dockerfile 한 편을 통째로 보고, 줄마다 왜 그렇게 했는지 풀어가자.

```dockerfile
# 1단계: 빌드
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./        # ① 의존성 목록 먼저
RUN npm ci                   # ② 설치 (캐시 핵심)
COPY . .                     # ③ 소스는 그 다음
RUN npm run build

# 2단계: 실행
FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/dist ./dist          # ④ 산출물만
COPY --from=build /app/node_modules ./node_modules
USER node                    # ⑤ 비-root로 실행
EXPOSE 3000
ENTRYPOINT ["node", "dist/server.js"]        # ⑥ exec 형태
```

## 5.2 ①②③ 캐시를 살리는 순서

4장의 핵심 규칙을 다시 적용한 부분이다.

의존성 목록(`package.json`)을 먼저 복사하고 설치한 뒤, **그 다음에** 소스 전체를 복사했다.
소스 코드만 바뀌었을 때 `npm ci` 캐시가 살아남아 라이브러리를 다시 안 깔게 하기 위해서다.

> 규칙: 자주 안 바뀌는 것(의존성)을 위에, 자주 바뀌는 것(소스)을 아래에.

여기에 하나 더 — **`.dockerignore`** 파일이다.
`.git`, `node_modules`, 빌드 산출물처럼 이미지에 들어갈 필요 없는 것들을 여기 적어 빌드에서 제외한다.
이걸 안 하면 불필요한 파일까지 복사되어 캐시가 자주 깨지고 이미지가 커진다.

```
# .dockerignore 예시
node_modules
.git
dist
*.log
```

## 5.3 ④ 멀티스테이지로 이미지 다이어트

`FROM`이 두 번 나온 게 보일 것이다.
4장에서 본 **멀티스테이지 빌드**다.

1단계(`AS build`)에서 컴파일·번들링을 다 하고, 2단계는 깨끗한 베이스에서 **완성된 결과물만**(`--from=build`) 가져온다.
빌드 도구가 최종 이미지에 안 들어가니 크기가 줄고, 공격 표면도 준다.

더 줄이고 싶으면 `scratch`(완전히 빈 베이스)나 distroless 이미지를 쓴다.
Go나 Rust처럼 결과물이 단일 실행 파일이면 효과가 특히 크다.

## 5.4 ⑤ 비-root로 실행

기본적으로 컨테이너 안 프로그램은 root로 돈다.
2장에서 봤듯 컨테이너 안 root는 위험할 수 있다 (커널을 공유하니까).

그래서 `USER node`처럼 **일반 사용자로 낮춰서** 실행하는 게 안전하다.
혹시 컨테이너가 뚫려도 공격자가 얻는 권한이 줄어든다.

여기에 더해 운영에서는 이런 것들도 쓴다.
- `--read-only` : 컨테이너의 루트 파일시스템을 읽기 전용으로 (변조 방지)
- 필요한 쓰기 경로만 `tmpfs`로 따로 허용

## 5.5 🔴 ⑥ ENTRYPOINT는 왜 대괄호([]) 형태인가

마지막 줄을 보자.

```dockerfile
ENTRYPOINT ["node", "dist/server.js"]   # 좋음 (exec 형태)
```

이걸 이렇게 쓸 수도 있다.

```dockerfile
ENTRYPOINT node dist/server.js          # 나쁨 (shell 형태)
```

둘은 겉보기엔 같아 보이지만 결과가 다르다.

대괄호가 없는 **shell 형태**로 쓰면, 리눅스가 먼저 셸(`/bin/sh`)을 띄우고 그 셸이 내 프로그램을 자식으로 실행한다.
그러면 2장에서 본 **PID 1**(첫 번째 프로세스)이 내 프로그램이 아니라 **셸**이 되어버린다.

이게 왜 문제냐면, `docker stop`으로 컨테이너를 멈출 때 보내는 종료 신호(SIGTERM)를 그 셸이 내 프로그램에 제대로 전달하지 못한다.
결과적으로 프로그램이 **깔끔하게 종료할 기회를 못 얻고** 강제로 죽는다 (정리 작업, 진행 중 요청 마무리 등을 못 함).

대괄호가 있는 **exec 형태**로 쓰면 셸을 거치지 않고 내 프로그램이 **직접 PID 1**이 되어 종료 신호를 바로 받는다.
그래서 항상 대괄호 형태로 쓰는 게 좋다.

이 PID 1 문제는 9장에서 더 깊이 파고든다.

## 5.6 헬스체크

컨테이너가 "떠 있다"와 "정상 동작한다"는 다르다.
프로세스는 살아 있지만 안에서 먹통일 수 있다.

`HEALTHCHECK`로 컨테이너 스스로 자기 상태를 점검하게 할 수 있다.

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:3000/health || exit 1
```

이러면 Docker가 주기적으로 이 명령을 돌려 컨테이너가 진짜 건강한지 확인한다.
8장의 Compose에서 "DB가 진짜 준비됐을 때 앱을 띄우기"에 이 헬스체크가 쓰인다.

## 5.7 정리

좋은 Dockerfile의 체크리스트로 정리한다.

| 목표 | 방법 |
|---|---|
| 빌드 빠르게 | 의존성 먼저 복사·설치, 소스는 나중 / `.dockerignore` |
| 이미지 작게 | 멀티스테이지, 가벼운 베이스(alpine·distroless) |
| 안전하게 | 비-root `USER`, `--read-only`, 최소 권한 |
| 신호 잘 받게 | `ENTRYPOINT`는 대괄호(exec) 형태 |
| 건강 확인 | `HEALTHCHECK` |

이제 이미지를 잘 만들 줄 안다.
다음은 컨테이너들을 **서로 연결**하는 이야기다.
6장에서 Docker 네트워킹으로 간다.
