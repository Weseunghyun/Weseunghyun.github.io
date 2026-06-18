---
title: "Docker 완전 정복 8장 — Docker Compose (여러 컨테이너 한 번에)"
date: 2026-06-17 23:51:00 +0900
categories: ["Docker", "Docker 완전 정복"]
tags: ["docker", "실전"]
---
지금까지 우리는 컨테이너를 `docker run` 명령으로 하나씩 띄웠다.
그런데 실제 서비스는 보통 여러 컨테이너의 묶음이다.
예를 들어 웹 앱 하나에 데이터베이스 하나, 캐시 하나.

이걸 `docker run`으로 하나씩 띄우려면 어떻게 될까?
네트워크를 만들고, 볼륨을 만들고, 환경 변수를 길게 적고, 순서를 맞춰서 컨테이너 셋을 따로따로 실행해야 한다.
명령이 길고, 매번 똑같이 치기 어렵고, 실수하기 쉽다.

**Docker Compose**는 이 묶음 전체를 **하나의 파일**로 적어두고 한 번에 다루는 도구다.

## 8.1 Compose 파일 한 편

`compose.yaml`이라는 파일에 서비스들의 구성을 적는다.

```yaml
services:
  api:
    build: .
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      DATABASE_URL: "postgres://db:5432/app"   # db라는 이름으로 접속
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:17
    volumes:
      - pgdata:/var/lib/postgresql/data         # 영속 데이터
    environment:
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

volumes:
  pgdata:
```

이 한 파일에 우리가 앞에서 배운 게 다 들어 있다.
- `api`와 `db` 두 **서비스**(컨테이너)
- `db`에 붙은 **볼륨**(7장)
- 환경 변수, 포트 퍼블리시(6장)

그리고 실행은 이 한 줄이면 된다.

```bash
docker compose up -d
```

이 명령 하나로 네트워크 생성, 볼륨 생성, 두 컨테이너 실행이 한꺼번에 일어난다.

## 8.2 🔴 이름으로 통신되는 이유

`api` 서비스의 환경 변수를 보자.

```
DATABASE_URL: "postgres://db:5432/app"
```

데이터베이스 주소로 IP가 아니라 `db`라는 **이름**을 썼다.
6장에서 "사용자 정의 네트워크를 만들어야 이름으로 통신된다"고 했는데, 여기서는 그런 명령을 따로 안 쳤다.

비밀은 이렇다.

> Docker Compose는 이 묶음을 위한 **사용자 정의 네트워크를 자동으로 만들어준다.** 그래서 서비스들이 서로의 **이름으로** 통신할 수 있다.

`api`는 `db`라는 서비스 이름만으로 데이터베이스를 찾아간다.
IP를 신경 쓸 필요가 없다.
이게 Compose가 멀티 컨테이너를 편하게 만드는 핵심이다.

## 8.3 🔴 기동 순서 — "시작"과 "준비"는 다르다

`api`에 이런 설정이 있었다.

```yaml
depends_on:
  db:
    condition: service_healthy
```

`depends_on`은 "db가 먼저 뜨고 나서 api를 띄워라"라는 순서 지정이다.
그런데 여기 함정이 하나 있다.

데이터베이스 컨테이너가 **"시작"된 것**과 **"접속을 받을 준비가 된 것"**은 다르다.
DB 프로세스가 막 떴어도, 내부 초기화가 끝나 쿼리를 받기까지는 몇 초가 더 걸린다.

단순히 `depends_on: db`만 쓰면 "db 컨테이너가 시작됐다"까지만 보장하고, "준비됐다"는 보장하지 않는다.
그래서 api가 너무 빨리 떠서 아직 준비 안 된 db에 접속하려다 실패할 수 있다.

이걸 막으려고 위 예제는 두 가지를 결합했다.
- `db`에 **healthcheck**를 달아 "진짜 준비됐는지" 확인하게 하고 (5장의 헬스체크)
- `api`는 `condition: service_healthy`로 "db가 **건강해진 뒤**" 뜨게 했다.

> 핵심: `depends_on`만으로는 "시작 순서"만 잡힌다. "준비 완료"까지 기다리려면 **healthcheck + service_healthy**를 함께 써야 한다.

## 8.4 자주 쓰는 Compose 명령

| 명령 | 하는 일 |
|---|---|
| `docker compose up -d` | 묶음 전체를 백그라운드로 실행 |
| `docker compose logs -f` | 전체 로그를 실시간으로 |
| `docker compose ps` | 서비스 상태 보기 |
| `docker compose down` | 묶음 정지·정리 |
| `docker compose down -v` | 🔴 볼륨까지 삭제 (데이터 날아감, 주의) |

특히 마지막 `-v`는 7장에서 경고한 대로 볼륨을 지우니, 운영에서 습관적으로 치지 않도록 조심한다.

## 8.5 Compose는 한 대까지, 그 너머는 쿠버네티스

Compose는 **한 서버 안에서** 여러 컨테이너를 다루는 데 최적이다.
개발 환경, 작은 서비스에는 이것으로 충분하다.

하지만 여러 서버에 걸쳐 컨테이너를 수십·수백 개 배치하고, 자동으로 장애 복구하고, 부하에 따라 늘리고 줄이는 일은 Compose의 영역이 아니다.
그건 **쿠버네티스(Kubernetes)** 같은 오케스트레이션 도구의 일이다.
(이 시리즈의 범위 밖이지만, 다음 학습 주제로 자연스럽게 이어진다.)

## 8.6 정리

이 장을 한 문장으로 줄이면 이렇다.

> Docker Compose는 여러 컨테이너 묶음을 **한 파일**로 정의해 `up` 한 번에 띄우는 도구이고, **사용자 정의 네트워크를 자동 생성**해 서비스 이름으로 통신되게 하며, 기동 순서는 **healthcheck + service_healthy**로 "준비 완료"까지 보장해야 한다.

이제 만들고, 연결하고, 묶는 법을 다 안다.
마지막으로, 실제 운영에서 **어디서 터지는가**를 모아 보자.
9장에서 Docker의 대표적인 함정들을 정리한다.
