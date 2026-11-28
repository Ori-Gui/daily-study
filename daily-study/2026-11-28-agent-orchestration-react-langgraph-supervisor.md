# [2026-11-28] AI 에이전트 오케스트레이션 — ReAct 루프 vs Plan-and-Execute, LangGraph 상태 그래프·Checkpointer·interrupt, Supervisor/Handoff 멀티 에이전트와 컨텍스트 격리·비용 설계

> **한 줄 핵심 요약**: 에이전트 오케스트레이션은 "프롬프트를 잘 쓰는 일"이 아니라 **루프의 종료 조건과 상태(state)의 소유자를 정하는 설계**입니다. ReAct는 매 스텝 모델이 다음 행동을 고르는 구조라 유연하지만 종료·중복·폭주를 코드로 강제해야 하고, Plan-and-Execute는 계획을 먼저 고정해 예측 가능성과 토큰 효율을 얻는 대신 재계획 비용이 생깁니다. 멀티 에이전트는 정확도 마법이 아니라 **컨텍스트 창을 쪼개 병렬로 토큰을 더 쓰는 방법**이며(Anthropic 내부 평가 +90.2%, 대신 토큰 약 15배), 그래서 도입 판단 기준은 "병렬화 가능한 독립 하위 작업이 충분한가 + 그 결과의 가치가 토큰 15배를 지불할 만한가"입니다.

---

## 1. 개념 및 동작 원리

### 1.1 "프롬프트 체인"과 "에이전트"의 경계

먼저 용어를 고정합니다. 실무에서 이 경계를 흐리게 쓰면 설계가 무너집니다.

| 구분 | 결정 주체 | 흐름 | 실패 시 대응 | 대표 예 |
| --- | --- | --- | --- | --- |
| 단일 호출(Completion) | 사람 | 입력 → 출력 1회 | 재호출 | 요약, 분류, 번역 |
| 프롬프트 체인(Workflow) | 사람(코드) | 정해진 순서 N개 호출 | 해당 단계 재시도 | 문서 추출 → 검증 → 저장 |
| 도구 호출(Function Calling) 1회 | 모델(도구 선택만) | 호출 → 도구 실행 → 최종 답 | 도구 오류 반환 후 재호출 | 날씨 조회 + 답변 |
| **에이전트(Agent)** | **모델(다음 행동 전체)** | **루프: 판단 → 행동 → 관찰 → 재판단** | **루프 안에서 스스로 수정** | 리서치, 코드 수정, 운영 자동화 |

- 결정적인 차이는 **제어 흐름(control flow)의 소유자**입니다. 워크플로는 사람이 쓴 `if/for`가 흐름을 정하고 LLM은 각 노드의 내용만 채웁니다. 에이전트는 **모델의 출력이 다음에 실행할 노드를 정합니다.** 그래서 에이전트의 품질 문제는 대부분 프롬프트가 아니라 **그래프 구조·종료 조건·도구 설계** 문제로 나타납니다.
- 2026년 기준으로 실무에서 가장 많이 쓰이는 형태는 양 극단이 아니라 **"워크플로 안에 제한된 자유도(에이전트)를 넣은 하이브리드"** 입니다. 예: 수집·정제·색인은 고정 파이프라인, "어떤 문서를 더 찾아볼지"만 에이전트 루프에 위임.

### 1.2 ReAct — 추론과 행동을 한 스트림에 섞는 기본 루프

ReAct(Yao et al., 2022)는 **Reasoning + Acting**의 합성어로, 모델이 `Thought → Action → Observation`을 반복하며 사고 흔적과 도구 실행 결과를 같은 컨텍스트에 누적하는 구조입니다.

```text
[ReAct 루프]

        ┌──────────────────────────────────────────────┐
        │  LLM (매 스텝 전체 히스토리를 다시 읽음)      │
        └───────────────┬──────────────────────────────┘
                        │ 출력: Thought + Action(tool, args)
                        ▼
        ┌──────────────────────────────────────────────┐
        │  런타임(내 코드): 도구 실행 · 권한 검사       │
        │  · 타임아웃 · 결과 크기 절단 · 로그 기록       │
        └───────────────┬──────────────────────────────┘
                        │ Observation(도구 결과)
                        ▼
        ┌──────────────────────────────────────────────┐
        │  종료 판정: Action == final_answer ?         │
        │            step >= MAX_STEP ?                │
        │            deadline 초과 ? token 예산 초과 ?  │
        └───────┬───────────────────────┬──────────────┘
                │ 계속                   │ 종료
                └──────► (다음 스텝)      └──► 최종 답/부분 결과/에러
```

최소 구현은 다음과 같습니다. 프레임워크 없이도 이 40줄이면 동작하고, **프레임워크를 쓰더라도 내부는 이 루프**라는 점을 알아야 디버깅이 됩니다.

```python
# react_loop.py — 뼈대만 남긴 ReAct 실행기
import json, time
from dataclasses import dataclass, field

@dataclass
class Step:
    index: int
    thought: str
    tool: str
    args: dict
    observation: str

def run_react(llm, tools: dict, question: str, *,
              max_steps: int = 8, time_budget_s: float = 45.0) -> dict:
    deadline = time.monotonic() + time_budget_s
    history, steps, seen = [], [], set()

    for i in range(max_steps):
        if time.monotonic() > deadline:
            return {"status": "time_budget", "steps": steps, "answer": None}

        msg = llm.chat(messages=history)          # Thought + Action 또는 final
        if msg.type == "final":
            return {"status": "ok", "steps": steps, "answer": msg.text}

        key = (msg.tool, json.dumps(msg.args, sort_keys=True, ensure_ascii=False))
        if key in seen:                            # 동일 호출 반복 = 루프 신호
            history.append({"role": "user",
                            "content": "동일한 도구 호출이 반복되었습니다. 다른 접근을 쓰거나 종료하세요."})
            continue
        seen.add(key)

        fn = tools.get(msg.tool)
        if fn is None:                             # 없는 도구 = 모델 환각
            observation = f"ERROR: unknown tool '{msg.tool}'. 사용 가능: {list(tools)}"
        else:
            try:
                observation = str(fn(**msg.args))[:4000]   # 결과 폭주 방지 절단
            except Exception as e:                         # 도구 오류도 관찰값으로 반환
                observation = f"ERROR: {type(e).__name__}: {e}"

        steps.append(Step(i, msg.thought, msg.tool, msg.args, observation))
        history += [{"role": "assistant", "content": msg.raw},
                    {"role": "tool", "content": observation}]

    return {"status": "step_limit", "steps": steps, "answer": None}
```

ReAct의 성질과 함정:

- **장점**: 예상치 못한 중간 결과에 적응합니다(검색 결과가 비면 쿼리를 바꾸는 식). 도구가 늘어나도 흐름을 다시 짤 필요가 없습니다.
- **단점 1 — 오류 누적(error compounding)**: 정확도 p인 스텝 10개면 전체 성공률은 대략 p¹⁰입니다. p=0.95여도 0.60입니다. 그래서 긴 작업은 스텝 수를 줄이거나(계획 고정) 중간 검증 노드를 넣습니다.
- **단점 2 — 컨텍스트 팽창**: 매 스텝 전체 히스토리를 다시 넣으므로 토큰이 **스텝 수에 대해 이차(O(n²))로** 늘어납니다(누적 입력 n × 평균 길이 n/2). 20스텝짜리 루프는 5스텝짜리보다 10배 이상 비쌉니다.
- **단점 3 — 종료를 모른다**: "충분히 찾았다"는 판단을 모델에게 맡기면 조기 종료(얕은 답)와 과잉 탐색(폭주)이 같이 나타납니다. Anthropic도 초기 버전에서 **단순 질의에 서브에이전트 50개를 생성**하는 실패를 겪고, 질의 복잡도별 작업량 규칙(단순 1개·3~10콜, 비교 2~4개·각 10~15콜, 복잡 10개 이상)을 프롬프트에 명시했습니다.
- **단점 4 — 결정성 부족**: 같은 질문에도 경로가 달라집니다. 재현이 필요한 업무(정산·리포트)는 경로를 고정해야 합니다.

### 1.3 Plan-and-Execute — 계획을 먼저 고정하고 실행은 싸게

계획(Planner)과 실행(Executor)을 분리합니다. Planner가 하위 작업 목록을 만들고, Executor가 각 작업을 (대개 도구 호출 1~2회로) 처리하며, 필요하면 Replanner가 계획을 수정합니다.

```text
[Plan-and-Execute]

 사용자 요청
     │
     ▼
 ┌─────────┐   계획(JSON 배열): ["A 조회", "B 계산", "C 작성"]
 │ Planner │──────────────────────────────────────────┐
 └─────────┘                                          │
     │                                                ▼
     │                                    ┌────────────────────┐
     │                                    │ Executor (스텝별)   │
     │                                    │  step1 → 결과1      │
     │                                    │  step2 → 결과2      │
     │                                    └─────────┬──────────┘
     │                                              │ 각 스텝 결과
     │        ┌────────────┐  계획 수정 필요?        │
     └───────►│ Replanner  │◄───────────────────────┘
              └─────┬──────┘
                    │ 아니오
                    ▼
              최종 합성(Writer) → 답변
```

| 항목 | ReAct | Plan-and-Execute |
| --- | --- | --- |
| 흐름 결정 시점 | 매 스텝 | 시작 시 1회(+재계획 시) |
| 컨텍스트 증가 | O(n²) (전체 히스토리 재입력) | O(n) (계획 + 스텝별 요약) |
| 예상치 못한 결과 대응 | 강함 | 약함(재계획이 있어야 함) |
| 재현성·감사 가능성 | 낮음 | 높음(계획이 산출물로 남음) |
| 비용 | 스텝 수에 급격히 증가 | 상대적으로 낮음 |
| 적합한 작업 | 탐색·디버깅·불확실한 조사 | 단계가 예측되는 업무(리포트 생성, 마이그레이션 점검) |
| 대표 실패 | 루프·과잉 탐색 | 잘못된 계획을 끝까지 수행 |

- 실무 절충안이 **"계획은 상위 모델, 실행은 하위 모델"** 입니다. Planner는 강한 모델 1회, Executor는 저렴한 모델 N회. 비용의 대부분이 Executor에 몰리기 때문에 모델 라우팅 효과가 큽니다.
- 계획은 반드시 **구조화 출력(JSON Schema)** 으로 받습니다. 자유 텍스트 계획은 파싱 실패와 스텝 누락을 만듭니다.
- 계획 검증 게이트를 둡니다: 허용된 도구만 참조하는가, 스텝 수 상한을 넘지 않는가, 같은 도구·같은 인자 반복이 없는가. 이 세 검사만으로 폭주의 상당 부분이 코드 레벨에서 차단됩니다.

### 1.4 상태 그래프(LangGraph) — 루프를 "그래프 + 체크포인트"로 바꾸는 이유

ReAct 루프를 `while` 문으로 직접 쓰면 (1) 사람 승인 대기로 멈췄다가 재개하기 어렵고 (2) 중간 실패 시 처음부터 다시 해야 하고 (3) 실행 경로를 시각화·테스트하기 어렵습니다. 상태 그래프는 이 셋을 해결합니다.

구성 요소:

| 요소 | 역할 | 실무에서의 의미 |
| --- | --- | --- |
| **State** | 노드들이 공유하는 타입드 딕셔너리 | 무엇이 "실행 상태"인지 명시. 대화·계획·결과·오류·예산을 여기 넣음 |
| **Reducer** | 상태 키별 병합 규칙(예: 메시지는 append) | 병렬 노드 결과를 합치는 규칙. 잘못 두면 덮어쓰기 사고 |
| **Node** | 함수(LLM 호출, 도구 실행, 검증) | 부수효과는 여기. 재실행 가능해야 함 |
| **Edge** | 고정 전이 / 조건부 전이(라우터) | 종료 조건, 반복 상한, 승인 분기를 코드로 강제 |
| **Checkpointer** | 매 super-step마다 상태 스냅샷 저장 | 재개·시간여행 디버깅·대화 메모리의 기반 |
| **thread_id** | 체크포인트를 묶는 키 | 대화/실행 1건의 식별자. 사용자·세션 단위로 설계 |
| **interrupt** | 노드 안에서 실행을 멈추고 외부 입력 대기 | 사람 승인(HITL), 추가 정보 요청 |

```python
# graph.py — 상태 그래프 + 사람 승인 interrupt (개념 구현)
from typing import Annotated, Literal, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import InMemorySaver   # 운영에서는 DB/Redis 기반 체크포인터
from langgraph.types import interrupt, Command

class State(TypedDict):
    messages: Annotated[list, add_messages]   # reducer: 누적
    plan: list[str]
    cursor: int
    approved: bool
    steps: int

def plan_node(state: State) -> dict:
    return {"plan": make_plan(state["messages"]), "cursor": 0, "steps": 0}

def act_node(state: State) -> dict:
    task = state["plan"][state["cursor"]]
    needs_approval = touches_money_or_publish(task)
    if needs_approval:
        # 승인 없이는 다음 노드로 못 감. 재개하면 이 노드는 처음부터 다시 실행됨
        ok = interrupt({"type": "approval", "task": task, "diff": preview(task)})
        if not ok:
            return {"approved": False}
    result = execute(task)
    return {"messages": [("tool", result)], "cursor": state["cursor"] + 1,
            "steps": state["steps"] + 1}

def route(state: State) -> Literal["act", "finish"]:
    if state.get("approved") is False:
        return "finish"
    if state["steps"] >= 8 or state["cursor"] >= len(state["plan"]):
        return "finish"
    return "act"

builder = StateGraph(State)
builder.add_node("plan", plan_node)
builder.add_node("act", act_node)
builder.add_edge(START, "plan")
builder.add_edge("plan", "act")
builder.add_conditional_edges("act", route, {"act": "act", "finish": END})
graph = builder.compile(checkpointer=InMemorySaver())

# 실행 1회차: act 노드에서 interrupt → 상태는 체크포인트에 저장되고 대기
cfg = {"configurable": {"thread_id": "user-42:run-20261128-01"}}
out = graph.invoke({"messages": [("user", "이번 주 실패 주문 정리해줘")]}, cfg)

# 재개: 같은 thread_id + Command(resume=...)
out = graph.invoke(Command(resume=True), cfg)
```

여기서 반드시 알아야 할 세 가지 동작:

1. **체크포인트는 "super-step" 단위**입니다. 노드 실행이 끝날 때마다 상태 스냅샷이 남으므로, 3번째 스텝에서 실패하면 2번째 스냅샷에서 **재시작**할 수 있습니다(`invoke(None, cfg)`가 마지막 체크포인트를 이어받는 패턴).
2. **interrupt 재개는 "그 노드를 처음부터 다시 실행"** 합니다. `interrupt()` 호출 지점부터가 아니라 **노드 함수 전체**가 재실행됩니다. 따라서 노드 안의 부수효과(메일 발송, 결제, 파일 쓰기)는 반드시 **승인 이후 구간에만** 두거나, 멱등키로 중복을 흡수해야 합니다. 이걸 모르고 "승인 전에 로그를 남기고 승인 후에 발송"처럼 짜면 로그가 두 번 남습니다.
3. **`thread_id`가 곧 재개의 열쇠**입니다. 새 `thread_id`는 빈 상태에서 시작하므로, 재시도 로직에서 ID를 새로 만들면 "이어하기"가 아니라 "새 작업"이 됩니다(11-19 노트의 Idempotency-Key와 같은 성질).

### 1.5 멀티 에이전트 토폴로지 — 무엇을 나누는가

멀티 에이전트는 "역할을 나눈 여러 프롬프트"가 아니라 **컨텍스트 창과 도구 권한을 나눈 여러 실행 주체**입니다. 주요 패턴:

| 패턴 | 구조 | 쓰는 이유 | 실패 양상 |
| --- | --- | --- | --- |
| **Orchestrator-Worker (Supervisor)** | 리드 에이전트가 하위 작업을 분해해 워커에 위임, 결과를 종합 | 병렬 탐색, 컨텍스트 격리 | 리드의 위임 지시가 모호하면 중복 작업·누락 |
| **Handoff (라우팅 이관)** | 대화·작업의 주도권을 다른 에이전트로 넘김(대개 1회성) | 상담 분야 전환, 전문가 라우팅 | 이관 루프(서로 넘기기), 컨텍스트 유실 |
| **Sequential / Pipeline** | A → B → C 고정 순서(작성 → 검토 → 교정) | 검증·교정이 필요한 산출물 | 앞 단계 오류가 그대로 전파 |
| **Hierarchical** | Supervisor 아래에 팀 리드, 그 아래 워커 | 대규모·다도메인 | 지연·비용 폭증, 추적 난이도 |
| **Debate / Critic** | 생성자와 비평자가 여러 라운드 교환 | 정답 검증이 어려운 서술형 | 합의 편향, 라운드 폭주 |
| **Network(peer)** | 에이전트끼리 자유롭게 도구처럼 호출 | 탐색적 문제 | **비권장** — 종료 보장이 없고 디버깅 불가 |

- **핵심 원칙: 하위 에이전트는 "도구"가 아니라 "격리된 컨텍스트를 가진 함수"** 로 취급합니다. 입력은 (목표, 출력 형식, 사용할 도구, 경계) 4개를 명시하고, 출력은 **요약된 구조화 결과**로 받습니다. 원문 전체를 돌려주면 컨텍스트 격리의 이점이 사라집니다.
- **Handoff와 Supervisor의 차이**는 주도권입니다. Handoff는 "이 대화는 이제 B가 담당"이고(대화 상태가 넘어감), Supervisor는 "A가 B에게 조회를 시키고 결과를 받아 계속 진행"입니다. 고객 응대처럼 대화 연속성이 중요한 곳은 Handoff, 리서치·배치는 Supervisor가 맞습니다.
- Anthropic의 리서치 시스템이 대표 사례입니다. 리드 에이전트가 계획을 **메모리에 저장**하고(컨텍스트 200k 초과 시 잘리기 때문), 서브에이전트를 병렬로 띄워 각자 자기 컨텍스트에서 탐색한 뒤 **압축된 소견만** 반환하게 합니다. 이 구조가 단일 에이전트 대비 내부 평가에서 **+90.2%** 를 냈고, 비용은 **채팅 대비 약 15배**(단일 에이전트는 약 4배)였습니다. 성능 분산의 95%를 (토큰 사용량·도구 호출 수·모델 선택) 세 요인이 설명했고, 그중 토큰 사용량만 80%를 설명했습니다. 즉 **"더 많이, 병렬로, 자기 컨텍스트에서"** 가 성능의 실체입니다.

### 1.6 멀티 에이전트가 손해인 경우 (도입 판정)

| 조건 | 단일 에이전트 | 멀티 에이전트 | 판정 |
| --- | --- | --- | --- |
| 하위 작업이 서로 독립·병렬 가능 | 순차로 느림 | 병렬로 단축 | ✅ 멀티 |
| 정보량이 단일 컨텍스트를 초과 | 잘림·망각 | 분산 수집 | ✅ 멀티 |
| 도구가 많고 권한이 다름 | 도구 선택 혼동 | 역할별 도구 격리 | ✅ 멀티 |
| 하위 작업 간 의존이 강함(앞 결과가 다음 입력) | 자연스러움 | 조정 오버헤드 | ❌ 단일/워크플로 |
| 코드 수정처럼 공유 컨텍스트가 필수 | 일관성 유지 | 충돌·중복 수정 | ❌ 단일 |
| 답이 짧고 정형(분류·추출) | 1콜 | 3~15배 비용 | ❌ 단일 |

Anthropic도 "코딩 작업은 리서치보다 병렬화 가능한 하위 작업이 적고, 에이전트가 실시간 위임·조정을 잘하지 못한다"고 명시했습니다. **"멀티 에이전트가 최신 트렌드니까"는 도입 근거가 되지 못합니다.**

---

## 2. 실무 구현: Spring Boot에서 오케스트레이터 직접 만들기

프레임워크 없이 Java로 루프를 소유하는 편이 나은 경우가 많습니다. 이유는 (1) 이미 Spring 트랜잭션·인증·관측 스택이 있고 (2) LLM 호출이 곧 외부 I/O라 **타임아웃·서킷브레이커·재시도·비동기 실행**을 기존 인프라로 처리하고 싶기 때문입니다.

### 2.1 패키지 구조

```text
com.example.agent
├── api/            AgentController            # POST /api/agent/runs (202 Accepted)
├── orchestrator/   AgentLoop, StepBudget, ToolRouter, PromptBuilder
├── tool/           OrderTools, DocSearchTools  # @Tool 메서드 + 권한 애노테이션
├── state/          AgentRun, AgentStep (JPA)   # 실행 상태 영속화
├── worker/         AgentRunWorker              # @Async / Kafka consumer
└── observe/        AgentTrace (Micrometer)
```

### 2.2 스텝 상한·예산·중복 차단을 코드로 강제한 루프

```java
// orchestrator/AgentLoop.java
// 의존성: org.springframework.ai:spring-ai-bom (버전은 팀 BOM에서 고정)
//         spring-boot-starter-web, spring-boot-starter-data-jpa, micrometer
package com.example.agent.orchestrator;

import java.time.*;
import java.util.*;
import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.model.ChatResponse;
import org.springframework.stereotype.Service;

@Service
public class AgentLoop {

    private static final int MAX_STEPS = 8;
    private static final Duration STEP_TIMEOUT = Duration.ofSeconds(20);
    private static final Duration RUN_BUDGET   = Duration.ofSeconds(60);
    private static final int MAX_OBSERVATION_CHARS = 4_000;

    private final ChatClient chatClient;     // 도구 콜백은 빌더에서 등록
    private final ToolRouter toolRouter;
    private final RunStore runStore;         // agent_run / agent_step 저장

    public AgentLoop(ChatClient.Builder builder, ToolRouter toolRouter, RunStore runStore) {
        this.chatClient = builder.build();
        this.toolRouter = toolRouter;
        this.runStore = runStore;
    }

    public RunResult run(RunContext ctx, String question) {
        Instant deadline = Instant.now().plus(RUN_BUDGET);
        List<StepLog> log = new ArrayList<>();
        Set<String> seenCalls = new HashSet<>();   // 중복 호출 감지
        List<org.springframework.ai.chat.messages.Message> messages = new ArrayList<>();
        messages.add(new org.springframework.ai.chat.messages.UserMessage(question));

        for (int step = 0; step < MAX_STEPS; step++) {
            if (Instant.now().isAfter(deadline)) {
                return RunResult.budgetExceeded(ctx.runId(), log);   // 부분 결과 반환
            }

            ChatResponse res = chatClient.prompt()
                    .messages(messages)
                    .toolCallbacks(toolRouter.callbacksFor(ctx.principal()))  // 권한별 도구 주입
                    .call()
                    .chatResponse();

            var output = res.getResult().getOutput();
            if (!output.hasToolCalls()) {                 // 최종 답변
                runStore.finish(ctx.runId(), output.getText(), log);
                return RunResult.ok(output.getText(), log);
            }

            var call = output.getToolCalls().get(0);
            String callKey = call.name() + ":" + toolRouter.normalize(call.arguments());
            if (!seenCalls.add(callKey)) {                // 같은 호출 반복 = 루프 신호
                messages.add(new org.springframework.ai.chat.messages.UserMessage(
                        "같은 도구를 같은 인자로 이미 호출했습니다. 다른 접근을 쓰거나 종료하세요."));
                continue;
            }

            String observation;
            try {
                observation = toolRouter.execute(ctx.principal(), call.name(), call.arguments())
                        .substring(0, MAX_OBSERVATION_CHARS);
            } catch (Exception e) {
                observation = "ERROR: " + e.getClass().getSimpleName() + " - " + e.getMessage();
            }

            StepLog entry = new StepLog(step, call.name(), call.arguments(), observation);
            log.add(entry);
            runStore.appendStep(ctx.runId(), entry);       // 감사·재개용 영속화
            messages.add(new org.springframework.ai.chat.messages.AssistantMessage(output.getText()));
            messages.add(new org.springframework.ai.chat.messages.ToolResponseMessage(List.of(
                    new org.springframework.ai.chat.messages.ToolResponseMessage.ToolResponse(
                            call.id(), call.name(), observation))));
        }
        return RunResult.stepLimit(ctx.runId(), log);
    }
}
```

포인트:

- **도구 목록을 권한별로 주입**합니다(`callbacksFor(principal)`). 사용자 A에게 없는 도구는 애초에 스키마에 없으므로 "호출 거부"를 판단할 필요가 없습니다. 권한 검사는 런타임에도 **도구 실행 직전에 한 번 더** 합니다(모델이 스키마를 환각할 수 있음).
- **도구 예외를 문자열로 바꿔 관찰값에 넣습니다.** 예외를 던져 루프를 죽이면 모델이 스스로 수정할 기회를 잃습니다. 대신 같은 도구·같은 인자로 3회 실패하면 **도구 서킷브레이커**를 열어 하드 실패시킵니다.
- **관찰값은 반드시 절단**합니다. 검색 결과 200KB를 그대로 넣으면 다음 스텝 컨텍스트가 폭발하고 비용이 배로 듭니다. 절단 사실도 관찰값에 명시(`truncated: true`)해 모델이 "더 좁혀 물어보게" 만듭니다.
- **비동기 실행**: 위 작업은 수십 초가 걸릴 수 있으므로 `POST /api/agent/runs`는 **202 Accepted + runId**만 반환하고, 실행은 `@Async`나 Kafka consumer가 맡습니다(11-16 노트의 at-least-once 소비 + 멱등 소비 조합). 상태는 `agent_run.status`(QUEUED/RUNNING/WAITING_APPROVAL/SUCCEEDED/FAILED/CANCELED)로 노출합니다.

### 2.3 실행 상태 모델 (JPA)

```java
// state/AgentRun.java
package com.example.agent.state;

import jakarta.persistence.*;
import java.time.Instant;

@Entity
@Table(name = "agent_run",
       uniqueConstraints = @UniqueConstraint(name = "uk_agent_run_idem",
                                             columnNames = {"user_id", "idempotency_key"}))
public class AgentRun {

    public enum Status { QUEUED, RUNNING, WAITING_APPROVAL, SUCCEEDED, FAILED, CANCELED }

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 36)
    private String runId;                      // 외부 노출용 UUID

    @Column(nullable = false) private String userId;
    @Column(nullable = false, length = 100) private String idempotencyKey;
    @Column(nullable = false) private String threadId;   // 재개 단위

    @Enumerated(EnumType.STRING) @Column(nullable = false)
    private Status status = Status.QUEUED;

    @Column(length = 2000) private String goal;
    @Column(length = 4000) private String lastError;

    @Column(nullable = false) private int stepCount;
    @Column(nullable = false) private long inputTokens;
    @Column(nullable = false) private long outputTokens;

    @Column(columnDefinition = "json") private String planJson;   // Plan-and-Execute용
    @Column(nullable = false) private Instant createdAt = Instant.now();
    private Instant finishedAt;
}
```

`(user_id, idempotency_key)` 유니크 제약이 **중복 실행의 최종 방어선**입니다. 사용자가 "다시 실행"을 두 번 눌러도 run은 하나이고, 워커는 같은 run을 두 번 처리해도 스텝 로그를 append할 뿐 부수효과는 멱등키로 흡수됩니다.

### 2.4 계획을 구조화 출력으로 받기

```java
// orchestrator/Planner.java — 계획 스키마를 코드로 고정
public record PlanStep(int order,
                       String tool,          // 반드시 ToolRouter의 화이트리스트에 존재
                       String purpose,
                       List<String> dependsOn) {}

public record Plan(List<PlanStep> steps) {}

public Plan makePlan(String goal) {
    Plan plan = chatClient.prompt()
            .system("""
                    너는 계획 수립기다. 사용 가능한 도구만 사용해 실행 계획을 만든다.
                    규칙: ① 스텝은 최대 6개 ② 같은 도구·같은 목적 중복 금지
                          ③ 각 스텝은 검증 가능한 산출물을 낸다 ④ 병렬 가능하면 dependsOn을 비운다
                    """)
            .user(goal)
            .call()
            .entity(Plan.class);       // 구조화 출력 → 스키마 위반 시 파싱 실패로 즉시 감지
    validatePlan(plan);                // 화이트리스트·상한·중복 검사 후 예외
    return plan;
}
```

계획을 `entity(Plan.class)`로 받으면 **스키마 위반이 런타임 초기에 드러납니다**. 자유 텍스트 계획을 정규식으로 파싱하는 방식은 반드시 깨집니다.

---

## 3. 실무 적용 및 주의사항 (Edge Cases)

### 3.1 폭주·무한 루프 — 4중 상한을 코드에 둔다

| 상한 | 값 예시 | 없을 때 나타나는 일 |
| --- | --- | --- |
| 스텝 수 | 8 | 같은 검색 반복, 종료 못 함 |
| 실행 시간 | 60s | 사용자 이탈, 워커 점유, LB 타임아웃 |
| 토큰/비용 | run당 200k input | 청구서 폭발, 예산 초과 알림 |
| 동일 호출 중복 | 2회 | "검색 → 빈 결과 → 같은 검색" 루프 |

- 상한 도달 시 **예외가 아니라 "부분 결과 + 상태 코드"** 로 반환합니다. `status: STEP_LIMIT`과 지금까지의 스텝 로그를 주면 사용자·운영자가 다음 행동을 정할 수 있습니다.
- 스텝 상한은 **모델에게도 알려줍니다**("남은 스텝 3개, 지금 종료를 준비하세요"). 남은 예산을 프롬프트에 넣으면 마무리 품질이 크게 달라집니다.
- Redis로 run 단위 락(`SET run:{id} NX PX 90000`)을 잡아 **같은 run의 동시 실행**을 막습니다. 락 TTL은 실행 예산보다 길게 잡아야 중간에 풀려 중복 실행되지 않습니다.

### 3.2 쓰기 도구는 "재시도되는 API"로 설계한다

에이전트 루프는 **같은 도구를 정당하게 여러 번 호출**할 수 있고, 모델은 실패한 호출을 재시도합니다. 즉 쓰기 도구는 언제든 재실행될 수 있습니다.

- **읽기 도구**: 자유롭게 재시도 허용.
- **쓰기 도구**: `idempotencyKey`를 **모델이 아니라 런타임이** 생성(runId + stepOrder + tool)해서 넘깁니다. 모델에게 키 생성을 맡기면 매번 새 키를 만들어 멱등성이 사라집니다.
- **되돌릴 수 없는 행동**(결제, 메일 발송, 배포, 외부 공개)은 **사람 승인 게이트**를 둡니다. 이때 승인 요청에는 반드시 **실행될 인자의 미리보기(diff)** 를 붙입니다. "승인하시겠습니까?"만 보여주는 승인은 승인이 아닙니다.
- 11-23 노트의 **Agents Rule of Two**(비신뢰 입력 · 민감 데이터 · 외부 통신 중 둘 이상을 한 실행에 묶지 않기)와 같은 원칙이 여기에도 적용됩니다. 오케스트레이터가 여러 도구를 조합할수록 이 삼각형이 쉽게 완성됩니다.

### 3.3 부분 실패와 오류 전파

```text
[서브에이전트 실패 처리 정책]

워커 실패
 ├─ 일시 오류(429/5xx/타임아웃) → 지수 백오프 2회 재시도 → 성공 시 계속
 ├─ 결정적 오류(400/스키마 위반) → 즉시 실패 기록, 다른 워커 계속
 └─ 최종 실패 → 리드에게 "실패 사유 + 부분 결과" 반환
                └─ 리드 판단: (a) 계획 수정 후 재시도 (b) 누락 표시하고 종합
                               (c) 전체 실패로 종료
```

- **전체를 롤백하지 않습니다.** 리서치는 5개 중 4개 성공이면 4개로 답하고 "X 영역은 확인 실패"를 명시하는 편이 낫습니다. 반면 정산·주문처럼 원자성이 필요한 작업은 워크플로로 짜고 실패 시 보상 트랜잭션(11-25 노트)을 씁니다.
- **실패도 관찰값**입니다. "이 도구가 3회 실패했다"를 컨텍스트에 남겨야 리드가 우회합니다.
- 재시도는 **워커 단위**로만 합니다. 리드까지 재시도하면 이미 성공한 워커의 부수효과가 중복됩니다.

### 3.4 컨텍스트 오염과 신뢰 경계

- 서브에이전트가 반환한 결과는 **외부 입력**입니다. 그 안에 "이전 지시를 무시하고 ~하라"가 들어 있을 수 있습니다(11-23 노트의 프롬프트 인젝션). 리드는 서브 결과를 **데이터 블록으로 격리**해 넣고, 그 안의 지시를 실행하지 않도록 시스템 프롬프트에 명시합니다.
- **요약이 곧 손실**입니다. 컨텍스트 압축은 토큰을 줄이지만 숫자·ID·인용을 지웁니다. 금액·주문번호·법령 조항처럼 정확성이 필요한 값은 요약하지 말고 **구조화 필드로 별도 보존**합니다.
- **메모리를 무한히 append하지 않습니다.** 대화 메모리는 최근 N턴 + 요약 1개로 유지하고, 원문은 DB에 두고 필요 시 검색(RAG, 11-14 노트)으로 되살립니다.

### 3.5 비용과 지연 설계

```text
[예시 계산 — 리서치 1건, 가정치]

단일 에이전트:  20k input × 6스텝 ≈ 120k 토큰 (히스토리 재입력 때문에 이차 증가)
멀티(리드1+워커4): 리드 15k×3 + 워커 4 × 25k = 145k
                  → 병렬 실행으로 벽시계 시간은 오히려 단축
Anthropic 보고 기준: 에이전트 ≈ 채팅의 4배, 멀티 에이전트 ≈ 15배
```

- **지연은 "스텝 수 × 모델 지연"의 합**이고, 병렬화는 여기서만 이깁니다. 도구 호출 대기(I/O)가 길면 병렬 도구 호출이 가장 큰 효과를 냅니다(Anthropic은 병렬 호출로 복잡 질의 리서치 시간을 최대 90% 단축).
- **모델 라우팅**: 분류·추출·요약은 소형 모델, 계획·최종 합성은 대형 모델. 라우팅만으로 비용 50~80% 절감이 흔합니다.
- **프롬프트 캐싱**: 시스템 프롬프트·도구 스키마·긴 문서를 앞쪽에 고정 배치해 캐시 히트를 노립니다. ReAct 루프처럼 같은 접두부를 반복하는 구조에서 효과가 큽니다.
- **비동기 + 스트리밍**: 사용자에게는 스텝 진행(계획 수립 → 검색 중 → 작성 중)을 SSE로 흘려 체감 지연을 줄입니다. 60초 침묵은 기능 결함으로 인식됩니다.

### 3.6 평가 — 최종 답만 보면 안 된다

| 평가 층 | 측정 | 도구 |
| --- | --- | --- |
| 최종 답 | 정답률, 근거 일치율, 형식 준수 | 골든셋 + 룰 기반 + LLM-as-a-Judge |
| 궤적(trajectory) | 스텝 수, 불필요 도구 호출, 중복 호출, 종료 사유 분포 | 트레이스 로그 집계 |
| 도구 | 호출 성공률, p95 지연, 오류 유형 | APM |
| 비용 | run당 토큰·금액, 모델별 분포 | 메트릭 |

- **궤적 평가가 없으면 회귀를 못 잡습니다.** 프롬프트를 조금 바꿔 정답률은 같아도 스텝이 6 → 14로 늘면 비용이 2배가 됩니다. "평균 스텝 수", "상한 도달 비율", "중복 호출 비율"을 대시보드에 둡니다.
- 프롬프트 변경은 **골든셋 30~50건 + A/B**로만 반영합니다. 체감으로 바꾸면 다른 케이스가 조용히 깨집니다.
- 도구 설명도 평가 대상입니다. Anthropic은 도구 설명을 재작성하는 것만으로 작업 완료 시간이 **40% 감소**했다고 보고했습니다. 도구 설명은 문서가 아니라 **프롬프트**입니다.

### 3.7 안티패턴

| 안티패턴 | 왜 문제인가 | 대안 |
| --- | --- | --- |
| 에이전트를 마이크로서비스처럼 잘게 나눔 | 조정 오버헤드·토큰 폭증, 종료 보장 없음 | 단일 에이전트 + 도구로 시작, 필요할 때만 분리 |
| 프레임워크부터 고름 | 루프·상태·종료 조건을 이해 못 해 디버깅 불가 | 40줄 루프를 먼저 만들고 프레임워크로 이전 |
| 자유 텍스트로 계획·도구 인자 파싱 | 파싱 실패, 누락, 인젝션 | 구조화 출력(JSON Schema) |
| "충분히 알았으면 종료해"만 신뢰 | 조기 종료·과잉 탐색 동시 발생 | 코드 상한 + 남은 예산 프롬프트 주입 |
| 도구 결과 무절단 주입 | 컨텍스트 폭발·비용 급증 | 절단 + 절단 사실 명시 + 페이지네이션 도구 |
| 승인 게이트를 프롬프트로만 | 모델이 스스로 승인 판단 | 런타임 강제(interrupt/상태 머신) |
| 스텝 로그 미저장 | 장애 원인 추적 불가, 재개 불가 | agent_step 테이블 + traceId |

### 3.8 실전 도입 체크리스트

- [ ] 루프 소유자가 코드인가(모델이 아닌가)? 종료 조건 4종이 모두 있는가?
- [ ] 스텝·시간·토큰·중복 상한이 설정값으로 분리되어 있는가?
- [ ] 쓰기 도구에 런타임 생성 멱등키가 있는가? 되돌릴 수 없는 행동에 승인 게이트가 있는가?
- [ ] 도구 권한이 (프롬프트가 아니라) 실행 계층에서 강제되는가?
- [ ] 실행 상태가 DB에 남아 재개·감사가 가능한가? `thread_id`가 안정적인가?
- [ ] interrupt 재개 시 노드 재실행을 견디는가(부수효과 위치·멱등성)?
- [ ] 궤적 지표(평균 스텝, 상한 도달률, 중복률, run당 비용)를 보고 있는가?
- [ ] 단일 에이전트로 충분한 작업을 멀티로 만들고 있지 않은가?

---

## 4. 기술 면접 / 실전 예상 질문 & 답변 팁

- **Q. ReAct와 Plan-and-Execute의 차이와 선택 기준은?**
  - **A.** ReAct는 매 스텝 모델이 다음 행동을 정해 예상치 못한 중간 결과에 적응하지만, 전체 히스토리를 재입력하므로 토큰이 스텝 수에 대해 이차로 늘고 오류가 누적되며 종료를 모릅니다. Plan-and-Execute는 계획을 먼저 고정해 컨텍스트 증가가 선형이고 재현·감사가 쉬운 대신, 계획이 틀리면 재계획이 필요합니다. **탐색적·불확실한 작업은 ReAct, 단계가 예측되는 업무는 Plan-and-Execute**, 실무는 "대형 모델로 계획 1회 + 소형 모델로 스텝 실행 + 게이트 검증"의 절충을 씁니다.

- **Q. 에이전트가 무한 루프에 빠지는 것을 어떻게 막는가?**
  - **A.** 프롬프트가 아니라 런타임에서 막습니다. ① 스텝 상한 ② 실행 시간 예산 ③ 토큰/비용 예산 ④ (도구, 정규화 인자) 해시로 중복 호출 감지 ⑤ 같은 도구 3회 연속 실패 시 서킷브레이커 ⑥ 남은 예산을 프롬프트에 주입해 마무리를 유도. 상한 도달은 예외가 아니라 **부분 결과 + 종료 사유**로 반환해 운영자가 판단하게 합니다.

- **Q. 멀티 에이전트는 항상 단일 에이전트보다 좋은가?**
  - **A.** 아닙니다. Anthropic 내부 평가에서 리드+서브에이전트 구조가 단일 대비 +90.2%였지만 토큰은 채팅의 약 15배(단일 에이전트는 4배)였습니다. 성능 향상의 실체는 **병렬로 더 많은 토큰을 쓰는 것**입니다. 따라서 병렬화 가능한 독립 하위 작업이 많고, 정보량이 단일 컨텍스트를 넘고, 결과의 가치가 비용을 상회할 때만 정당화됩니다. 코드 수정이나 의존이 강한 작업은 공유 컨텍스트가 필요해 단일 에이전트가 낫습니다.

- **Q. Supervisor와 Handoff의 차이는?**
  - **A.** Supervisor는 리드가 하위 에이전트에 **작업을 위임하고 결과를 받아** 계속 진행하는 구조(주도권 유지)이고, Handoff는 대화·작업의 **주도권 자체를 다른 에이전트로 이관**하는 구조입니다. 리서치·배치는 Supervisor, 고객 상담의 분야 전환은 Handoff가 맞습니다. Handoff는 서로 넘기는 루프가 생기지 않도록 이관 이력을 상태에 남기고 횟수를 제한해야 합니다.

- **Q. 에이전트 실행 상태를 왜 DB에 저장하는가?**
  - **A.** 세 가지 때문입니다. ① **재개**: 중간 실패 시 마지막 성공 스텝에서 이어서 실행(체크포인트). ② **사람 승인**: 승인 대기 동안 프로세스가 살아 있을 수 없으므로 상태를 저장하고 외부 입력으로 재개. ③ **감사·평가**: 어떤 도구를 어떤 인자로 몇 번 호출했는지 남겨야 장애 원인과 궤적 지표를 볼 수 있습니다.

- **Q. 사람 승인(HITL)을 넣을 때 주의할 점은?**
  - **A.** 승인 화면에 **실행될 인자의 미리보기**를 보여줘야 하고, 승인 이후 구간에만 부수효과를 두어야 합니다. LangGraph처럼 재개 시 노드를 처음부터 다시 실행하는 구조에서는 승인 전 코드가 두 번 실행되므로, 로그·발송 같은 부수효과는 승인 뒤로 옮기거나 멱등키로 흡수해야 합니다.

- **Q. 도구(function calling)와 에이전트의 차이는?**
  - **A.** 도구 호출은 모델이 **한 번** 도구를 골라 실행하고 답하는 것이고, 에이전트는 그 판단·실행·관찰을 **루프**로 반복하며 스스로 다음 행동을 정합니다. 제어 흐름의 소유자가 사람(코드)인지 모델인지가 경계입니다. 실무에서는 워크플로 안에 제한된 루프만 넣는 하이브리드가 가장 안정적입니다.

- **Q. MCP와 오케스트레이션의 관계는?**
  - **A.** MCP는 도구·데이터를 **표준 프로토콜로 노출**하는 계층이고, 오케스트레이션은 그 도구들을 **어떤 순서·권한·예산으로 호출할지** 정하는 상위 계층입니다. MCP 서버가 늘어나면 도구 스키마·설명 품질이 곧 성능이 되므로(11-17 노트), 오케스트레이터는 도구를 역할별로 격리해 노출하고 실행 계층에서 권한을 강제해야 합니다.

- **Q. (프로젝트 꼬리질문) AGZAK의 RAG를 에이전트로 확장한다면?**
  - **A.** 정적 검색 1회(RAG)를 "질의 재작성 → 다중 검색 → 근거 검증 → 부족 시 재검색"의 **제한된 ReAct 루프**로 바꾸는 것이 1단계입니다. 이때 ① 검색 스텝 상한 3회, ② 동일 쿼리 중복 차단, ③ 인용 없는 문장 금지 검증 노드, ④ 스텝 로그 저장(궤적 평가)을 함께 넣어야 합니다. 멀티 에이전트로 바로 가지 않는 이유는, 문서 검색이 단일 컨텍스트 안에서 충분히 처리되고 병렬화 이득이 토큰 15배를 정당화하지 못하기 때문입니다.

---

## 5. 정리

- 에이전트의 본질은 **루프 + 상태 + 종료 조건**이고, 품질 문제의 대부분은 프롬프트가 아니라 이 셋의 설계에서 나옵니다.
- ReAct는 적응력, Plan-and-Execute는 예측 가능성과 비용 효율, 상태 그래프는 **재개·승인·관측**을 줍니다. 셋은 대체재가 아니라 조합 대상입니다.
- 멀티 에이전트는 **컨텍스트 격리와 병렬 토큰 지출**이라는 비용을 지불하고 성능을 사는 거래입니다. 도입 전에 "병렬화 가능한 독립 작업이 있는가"와 "결과의 가치가 토큰 15배를 넘는가"를 먼저 계산합니다.
- 실무 방어선은 코드에 있습니다: 스텝·시간·토큰 상한, 중복 호출 감지, 런타임 멱등키, 승인 게이트, 권한별 도구 주입, 스텝 로그 영속화.
