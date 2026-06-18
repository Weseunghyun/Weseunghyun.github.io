---
title: "ClickHouse 완전 정복 16장 — AI 기능: 자연어로 묻고, 에이전트가 분석하는 ClickHouse"
date: 2026-06-13 23:43:00 +0900
categories: ["ClickHouse", "ClickHouse 완전 정복"]
tags: ["clickhouse", "심화"]
---
15장까지 우리는 ClickHouse를 *사람이 SQL을 쓰는 도구*로 다뤘다.
컬럼 저장, MergeTree, sparse index, 복제, 분산 — 전부 "내가 쿼리를 잘 짜면 빠르게 답이 온다"는 전제 위에 있었다.

이번 장은 그 전제를 한 칸 옆으로 민다.
"쿼리를 짜는 주체가 사람이 아니라 LLM이면 어떻게 되는가?"

2025~2026년 ClickHouse가 붙인 AI 기능들은 대부분 이 한 문장에서 출발한다.
즉 *분석가가 줄어드는 게 아니라, 분석을 요청하는 주체가 자연어를 쓰는 사람 + 그 사람을 대신하는 에이전트로 늘어난다*는 가정이다.

이 장은 신제품 카탈로그가 아니다.
공개적으로 확인되는 범위에서, 각 기능이 **무엇이고 → 어떻게 쓰는지 → 어디까지가 사실인지**를 차분히 정리한다.
빠르게 바뀌는 영역이라 "오늘(2026-06-13) 기준"이라는 단서를 머리에 달고 읽자.

## 첫 원리 — 왜 분석 DB에 AI가 붙는가

ClickHouse 측은 "agent-facing analytics"라는 표현을 쓴다.
에이전트(LLM)가 사람보다 훨씬 많은 쿼리를 던진다는 관찰이다.
공식 자료는 에이전트가 사람의 "10배에서 100배"에 달하는 쿼리 볼륨을 만든다고 인용한다 (공식 블로그 기준).

이게 왜 ClickHouse와 잘 맞을까.
LLM은 질문을 던질 때 "일단 넓게 훑고, 결과 보고 다시 좁히는" 식으로 탐색한다.
사람이라면 한 번 고민해서 한 방에 잘 짜려 하지만, 에이전트는 값이 싸니까 여러 번 묻는다.
그러면 *대량의 탐색적 집계 쿼리를 싸고 빠르게 받아주는 엔진*이 필요해진다.
그게 바로 2장에서 본 컬럼 저장 + 벡터화 + 병렬의 ClickHouse가 잘하는 일이다.

비유를 하나 들자 — 이건 어디까지나 이해를 돕는 비유다.
기존 BI는 "정성껏 쓴 편지"를 주고받는 우편이다.
사람이 질문을 곱씹어 한 통 보내고, 답을 기다린다.
반면 에이전트 분석은 "채팅"에 가깝다.
짧은 질문을 빠르게 연달아 던지고, 답을 보고 즉시 다음을 묻는다.
우편 시대의 우체국으로 채팅 트래픽을 받으면 줄이 막힌다.
ClickHouse는 자기를 "채팅 트래픽도 받아내는 창구"로 포지셔닝하고, 그 위에 자연어 인터페이스를 얹는 중이다.

실제 내부 구조와 정확한 라우팅은 이 비유보다 훨씬 평범하다 — 그냥 LLM이 SQL을 만들어 ClickHouse에 SELECT를 날리는 것이다.
아래에서 그 경로를 하나씩 본다.

## 1. clickhouse client 내장 자연어→SQL (가장 손에 잡히는 것)

가장 먼저 만질 수 있는 건 **클라이언트에 내장된 자연어→SQL**이다.
ClickHouse 25.7부터 `clickhouse client`와 `clickhouse-local`에 자연어로 질문하면 SQL을 만들어주는 기능이 들어왔다 (공식 문서 기준).

쓰는 법은 단순하다.
질문 앞에 `??`를 붙이면 그 줄을 SQL이 아니라 "자연어 질문"으로 해석한다.
내부적으로 LLM이 `list_databases`, `list_tables_in_database`, `get_schema_for_table` 같은 도구를 호출해 먼저 스키마를 탐색하고, 그 정보를 바탕으로 SQL을 생성한다.

쓰기 전에 API 키를 환경변수로 넘긴다.

```sql
export ANTHROPIC_API_KEY=...
-- 또는
export OPENAI_API_KEY=...
```

키를 넣고 클라이언트에 들어가면 이렇게 묻는다.

```sql
?? 지난 7일 동안 가장 트래픽이 많았던 상위 10개 페이지를 보여줘
```

그러면 LLM이 스키마를 훑은 뒤 그에 맞는 `SELECT ... GROUP BY ... ORDER BY ... LIMIT 10` 형태의 쿼리를 만들어 보여준다.
Anthropic과 OpenAI를 모두 지원한다 (공식 문서 기준).

여기서 두 가지를 정직하게 짚자.
첫째, 이 기능은 **대화 히스토리를 유지하지 않는다** (공식 문서 기준).
즉 "방금 그거에서 모바일만 빼줘" 같은 후속 질문이 앞 맥락을 이어받는다고 가정하면 안 된다.
각 질문은 독립적으로 스키마를 다시 탐색해 SQL을 만드는 데 가깝다.
둘째, API 키는 환경변수가 흔한 경로일 뿐 유일한 경로는 아니다 — 명시적 설정으로도 줄 수 있다.

## 2. 공식 MCP 서버 — 에이전트가 ClickHouse에 붙는 표준 통로

다음은 **MCP(Model Context Protocol) 서버**다.
MCP는 LLM 도구(예: Claude, IDE의 AI 어시스턴트)가 외부 시스템에 표준 방식으로 붙도록 하는 프로토콜이고, ClickHouse는 공식 MCP 서버 `mcp-clickhouse`를 오픈소스로 제공한다 (GitHub 공식 repo 기준).

특징을 정리하면 이렇다.

| 항목 | 내용 |
|---|---|
| 언어/설치 | Python, `pip` / `uv` 로 설치 |
| 기본 동작 | 읽기 전용(쓰기 비활성)이 기본 |
| 쓰기 허용 | `CLICKHOUSE_ALLOW_WRITE_ACCESS=false`(기본)로 강제, 풀려면 환경변수를 바꿔야 함 |
| 주 쿼리 도구 | `run_query` |
| chDB 옵션 | `run_chdb_select_query` 로 chDB(임베디드 ClickHouse) 쿼리 지원 |

여기서 흔한 오해 두 가지를 교정하자.
하나, "읽기 전용"을 SQL 세션 설정인 `readonly=1`로 거는 게 아니다.
공개 repo 기준으로 읽기 전용 강제는 `CLICKHOUSE_ALLOW_WRITE_ACCESS=false` **환경변수**로 이뤄진다.
둘, 과거에 돌던 "도구 이름은 `run_select_query`" 같은 정보는 현재 공식 repo 페이지에서 확인되지 않는다 — 지금 주 도구명은 `run_query`다.
버전별로 도구명이 바뀌어 온 정황은 있지만, 릴리스 노트 같은 별도 1차 근거 없이 단정하지는 않겠다.

얼마나 쓰이느냐도 자주 부풀려지는 지점이다.
ClickPy(ClickHouse 자체 PyPI 통계) 기준 `mcp-clickhouse`의 누적 다운로드는 약 120만 건이다 (ClickPy 기준).
간혹 보이는 "22만+" 같은 수치는 오래된 값이다.

## 3. AgentHouse — 공개 데모로 직접 만져보기

위 1·2번이 추상적으로 들린다면, ClickHouse가 띄워둔 공개 데모 **AgentHouse**가 가장 빠른 체험이다.

AgentHouse는 `llm.clickhouse.com`에서 돌아가는 인터랙티브 공개 데모로, ClickHouse의 실시간 분석을 Anthropic Claude(Sonnet) + ClickHouse MCP 서버와 결합해 자연어 질의를 시연한다 (공식 블로그 기준, 2025-04-24).
공개 데이터셋 37개가 올라가 있어, 자연어로 물으면 에이전트가 MCP를 통해 ClickHouse에 쿼리를 날리고 답을 정리해준다.

이게 중요한 이유는 앞의 "에이전트가 SQL을 만들어 SELECT를 날린다"는 흐름이 실제로 어떻게 보이는지를 손으로 확인할 수 있기 때문이다.
설치도, 키 발급도 없이 1·2번의 결합 결과를 눈으로 본다.

## 4. ClickHouse Cloud의 에이전틱 분석 — Ask AI와 Remote MCP

Cloud 쪽에는 관리형 기능이 더 붙는다.
2025-09-25 베타로 공개된 **Ask AI 에이전트**는 자연어로 쿼리·시각화·요약을 생성한다 (공식 블로그 기준).
문서 질문을 다루는 **Docs AI 서브에이전트**도 포함한다.

같이 나온 **Remote MCP 서버**는 ClickHouse Cloud에서 완전 관리형으로 동작하는 MCP 엔드포인트로, OAuth로 인증한다 (공식 블로그 기준).
2번의 self-managed `mcp-clickhouse`를 직접 띄우는 대신, Cloud가 호스팅해주는 버전이라고 보면 된다.

한 가지 톤을 낮춰 말할 부분이 있다.
이 Remote MCP를 "read-only SELECT만"이라고 딱 잘라 말하긴 어렵다.
공개 블로그는 "분석 맥락을 가져온다(retrieve analytical context)"는 식으로 서술하지, "SELECT로 한정된다"는 직접 표현을 쓰지 않는다.
방향은 읽기 중심이 맞지만, 정확한 권한 경계는 공개 자료만으로 단정하지 않는 게 안전하다.

## 5. ClickHouse Agents — 가장 최근(2026-06)의 관리형 에이전트

가장 최근 조각은 **ClickHouse Agents**다.
2026-06-09(Open House 2026 SF) 퍼블릭 베타로 공개된 Cloud 네이티브 에이전틱 분석 경험이다 (공식 블로그 기준).

공개된 구성 요소를 표로 정리하면 이렇다.

| 구성 | 내용 |
|---|---|
| 기본 모델 | Claude(Sonnet / Haiku) |
| 빌더 | 노코드(no-code) 에이전트 빌더 |
| 실행 | 샌드박스 코드 인터프리터(Bash / Python / JS) |
| 구조 | 멀티 에이전트 |
| 부가 | skills / memories / artifacts |

앞서 말한 "agent-facing analytics" — 에이전트가 사람의 10~100배 쿼리를 던진다는 관찰 — 이 ClickHouse Agents가 겨냥하는 시나리오다.
즉 분석 결과를 사람이 한 번 보는 게 아니라, 에이전트가 반복 질의하며 워크플로를 돌리는 것을 전제로 만든 기능이다.

## 6. 벡터 검색 — AI 워크로드를 위한 저장·검색 측면

AI 기능이라고 하면 자연어 인터페이스만 떠올리기 쉽지만, **벡터 검색**도 같은 줄기다.
LLM 애플리케이션의 RAG(검색 증강 생성)는 텍스트를 임베딩(숫자 벡터)으로 바꿔 저장하고, 질문 임베딩과 가까운 것을 찾아오는 일이다.

ClickHouse는 임베딩을 그냥 `Float` 배열 컬럼에 저장하고, 거리 함수로 가까운 벡터를 찾는다.
거리 함수는 `L2Distance`, `cosineDistance`, `dotProduct` 등을 쓴다 (공식 문서 기준).
근사 최근접 이웃(ANN) 검색은 보통 이런 형태다.

```sql
SELECT id, text
FROM documents
ORDER BY cosineDistance(embedding, [질문_임베딩]) ASC
LIMIT 5
```

여기서 `ORDER BY 거리함수 ... LIMIT N` 패턴이 핵심이다 — "가장 가까운 N개"를 정렬로 표현한다.

전체 스캔이 부담스러우면 인덱스를 건다.
ClickHouse 25.8부터 `vector_similarity` 인덱스(내부적으로 HNSW)를 쓸 수 있다 (공식 문서 기준).
양자화 기본값은 `bf16`이다 (공식 문서 기준).
정렬·거리 계산을 사람이 짜는 1장의 SQL과 똑같은 문법 위에서 한다는 게 포인트다 — 새 질의 언어가 아니라 익숙한 `ORDER BY ... LIMIT`로 벡터 검색을 표현한다.

## 7. 관측성 쪽 — HyperDX와 ClickStack

마지막으로, AI 시대의 운영을 떠받치는 관측성 조각도 짚자.
직접적인 "LLM 기능"은 아니지만, 에이전트·LLM 앱이 만들어내는 로그/트레이스를 받아내는 인프라라 같은 흐름에 있다.

ClickHouse는 2025-03-13에 **HyperDX**를 인수했다 (공식 블로그 기준).
HyperDX는 ClickHouse 위에 올라간 완전 오픈소스 관측성 플랫폼으로, UI와 세션 리플레이를 제공한다.

여기에 OTel(OpenTelemetry) Collector를 결합한 게 **ClickStack**이다.
ClickStack은 OTel Collector(수집) + ClickHouse(저장) + HyperDX(UI)로 구성된 OTel 네이티브 오픈소스 관측성 스택으로, 로그·메트릭·트레이스·세션 리플레이를 다룬다 (공식 블로그 기준).
Cloud 제공은 2025-08-06에 발표됐는데 — 이 시점은 일반 제공이 아니라 Private Preview였다는 점을 정직하게 덧붙인다.

## 큰 그림 — clickhouse.com/ai 의 묶음

위 조각들은 ClickHouse가 `clickhouse.com/ai` 페이지에서 "AI를 위한 플랫폼"으로 묶어 소개한다.
공개 페이지 기준으로 거기 나열되는 구성 요소는 다음과 같다 (공식 페이지 기준).

| 컴포넌트 | 역할(개략) |
|---|---|
| Chat Interface | 자연어 대화 인터페이스 |
| No-Code Agent Builder | 노코드 에이전트 빌더 |
| ClickHouse Assistant | SQL 작성·디버그 보조 |
| Sandboxed Code Interpreter | 샌드박스 코드 실행 |
| MCP Connectivity | MCP 연결 |
| Multi-agent Workflows / Sub-agents | 멀티 에이전트·서브에이전트 |
| Langfuse | LLM 관측성 |

여기서도 과장하지 않기 위해 한 가지 메모.
"ClickStack"이나 "Agent-Facing Analytics"는 이 `/ai` 페이지에서 독립 컴포넌트로 나열되지는 않는다 — ClickStack은 관측성 다이어그램 쪽, Agent-Facing Analytics는 개념으로만 등장한다.
그러니 위 7개를 "AI 플랫폼의 구성 요소", 나머지는 "관련 개념/제품"으로 구분해 기억하는 게 정확하다.

## "왜 이렇게?" — 한 발 물러서서

ClickHouse의 AI 전략을 한 문장으로 줄이면 이렇다.
*새 AI 데이터베이스를 만드는 게 아니라, 자기가 잘하는 빠른 OLAP 위에 "자연어로 묻는 입구"와 "에이전트가 붙는 표준 통로(MCP)"를 얹는다.*

자연어→SQL은 결국 1장의 SQL을 사람이 아니라 LLM이 쓰게 하는 것이고,
벡터 검색은 5장·11장의 정렬·필터 문법을 임베딩에 적용한 것이며,
MCP는 그 모든 걸 에이전트가 표준 방식으로 호출하게 하는 어댑터다.
관측성(HyperDX/ClickStack)은 그렇게 늘어난 트래픽을 다시 ClickHouse로 저장해 분석한다.

다시 말해 이 장의 기능들은 앞 15장에서 본 엔진의 *바깥에 새로 자란 인터페이스*이지, 엔진 자체를 바꾼 게 아니다.
그래서 2~13장을 이해한 사람일수록 이 AI 기능들이 어디서 빠르고 어디서 한계가 있는지를 더 잘 가늠한다.

빠르게 바뀌는 영역이라, 여기 적은 버전·날짜·도구명은 "2026-06-13 시점의 공개 자료"라는 단서를 항상 붙여 읽자.
17장과 18장에서는 다시 한발 물러나, ClickHouse 같은 컬럼 기반 OLAP가 어디서 왔고 전체 분석 DB 지형에서 어디에 서 있는지를 비교로 정리한다.

## 참고

- ClickHouse Docs — AI-powered SQL generation: https://clickhouse.com/docs/use-cases/AI/ai-powered-sql-generation
- ClickHouse PR #83314 (natural language to SQL): https://github.com/ClickHouse/ClickHouse/pull/83314
- ClickHouse Blog — Agentic analytics: Ask AI agent and Remote MCP server beta: https://clickhouse.com/blog/agentic-analytics-ask-ai-agent-and-remote-mcp-server-beta-launch
- ClickHouse MCP server (GitHub): https://github.com/ClickHouse/mcp-clickhouse
- ClickPy — mcp-clickhouse 통계: https://clickpy.clickhouse.com/dashboard/mcp-clickhouse
- ClickHouse Blog — ClickHouse Agents beta: https://clickhouse.com/blog/clickhouse-agents-beta
- ClickHouse Blog — AgentHouse demo (ClickHouse + LLM + MCP): https://clickhouse.com/blog/agenthouse-demo-clickhouse-llm-mcp
- ClickHouse Blog — ClickHouse acquires HyperDX: https://clickhouse.com/blog/clickhouse-acquires-hyperdx-the-future-of-open-source-observability
- ClickHouse Blog — Announcing ClickStack in ClickHouse Cloud: https://clickhouse.com/blog/announcing-clickstack-in-clickhouse-cloud
- ClickHouse Docs — Approximate Nearest Neighbor (vector_similarity) indexes: https://clickhouse.com/docs/engines/table-engines/mergetree-family/annindexes
- ClickHouse PR #53447 (vector similarity index): https://github.com/ClickHouse/ClickHouse/pull/53447
- ClickHouse Platform for AI: https://clickhouse.com/ai
