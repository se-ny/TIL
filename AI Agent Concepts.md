# TIL: AI Agent Concepts

> 날짜: 2026-06-17  
> 태그: `#AI` `#Agent` `#LangGraph` `#LLM` `#Python`

---

## 📌 AI 에이전트란?

**AI 에이전트(Agent)** 는 LLM이 단순히 답변을 생성하는 것을 넘어,
**스스로 판단하고, 도구를 사용하며, 목표를 달성하기 위해 여러 단계를 실행**하는 시스템이다.

```
일반 LLM 호출:
사용자 질문 → LLM → 답변 (1회성)

AI 에이전트:
목표 → LLM이 계획 수립 → 도구 실행 → 결과 관찰 → 재판단 → ... → 최종 답변
```

---

## 📌 에이전트 vs 단순 LLM 호출

| 구분 | 단순 LLM | AI 에이전트 |
|------|---------|------------|
| 동작 방식 | 질문 → 답변 (1회) | 목표 → 계획 → 실행 → 반복 |
| 외부 도구 사용 | ❌ | ✅ (검색, DB, API 등) |
| 자율적 판단 | ❌ | ✅ |
| 다단계 작업 | ❌ | ✅ |
| 예시 | 질문 답변, 번역 | 채용공고 수집 → 분석 → 저장 |

---

## 📌 에이전트의 핵심 구성 요소

```
┌─────────────────────────────────────────┐
│              AI 에이전트                  │
│                                         │
│  ┌─────────┐     ┌─────────────────┐   │
│  │   LLM   │────→│  Tool 선택 & 실행 │   │
│  │(두뇌/판단)│←────│  (행동 실행)     │   │
│  └─────────┘     └─────────────────┘   │
│       ↕                   ↕            │
│  ┌─────────┐     ┌─────────────────┐   │
│  │ Memory  │     │   Tools (도구)   │   │
│  │(상태 유지)│     │ 웹검색/DB/API 등 │   │
│  └─────────┘     └─────────────────┘   │
└─────────────────────────────────────────┘
```

### 1️⃣ LLM (두뇌)
- 상황을 판단하고 다음 행동을 결정
- "지금 뭘 해야 하는가?" 를 스스로 결정

### 2️⃣ Tools (도구)
- 에이전트가 실행할 수 있는 함수들
- 웹 검색, DB 조회, API 호출, 파일 읽기 등

### 3️⃣ Memory (메모리)
- 이전 단계의 결과를 기억
- 단기 메모리(대화 히스토리) / 장기 메모리(벡터 DB)

### 4️⃣ Planning (계획)
- 목표를 달성하기 위한 단계 수립
- ReAct, CoT 등의 추론 패턴 사용

---

## 📌 ReAct 패턴 (Reasoning + Acting)

에이전트가 가장 많이 사용하는 추론-행동 루프.

```
목표: "2024년 한국 AI 스타트업 채용 현황을 조사해줘"

[Thought]  채용 현황을 알려면 최신 정보를 검색해야겠다.
[Action]   web_search("2024 한국 AI 스타트업 채용")
[Observe]  검색 결과: "네이버, 카카오, 토스 AI팀 채용 중..."

[Thought]  더 구체적인 정보가 필요하다. 직무별로 검색해보자.
[Action]   web_search("2024 한국 AI 엔지니어 채용 요건")
[Observe]  검색 결과: "Python, FastAPI, LLM 경험 우대..."

[Thought]  충분한 정보가 모였다. 이제 정리해서 답변할 수 있다.
[Final Answer] "2024년 한국 AI 스타트업은..."
```

---

## 📌 Tool 정의 방법 (LangChain)

```python
from langchain.tools import tool
from langchain_community.tools import DuckDuckGoSearchRun

# 커스텀 Tool 정의
@tool
def get_job_postings(keyword: str) -> str:
    """채용공고를 검색하는 도구. keyword로 검색어를 입력받는다."""
    # 실제로는 DB 조회 또는 크롤링
    return f"{keyword} 관련 채용공고: FastAPI 개발자 5건, AI 엔지니어 3건"

@tool
def analyze_requirements(job_description: str) -> str:
    """채용공고의 요구사항을 분석하는 도구."""
    return f"필수 기술: Python, FastAPI / 우대사항: LLM 경험"

# 웹 검색 Tool
search = DuckDuckGoSearchRun()

# 도구 목록
tools = [get_job_postings, analyze_requirements, search]
```

---

## 📌 LangGraph로 에이전트 구현

**LangGraph** 는 에이전트의 흐름을 **그래프(노드 + 엣지)** 로 정의하는 프레임워크다.

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolExecutor
from typing import TypedDict, List

# 상태 정의
class AgentState(TypedDict):
    messages: List[dict]
    next_action: str

# 노드 정의
def call_llm(state: AgentState) -> AgentState:
    """LLM을 호출해서 다음 행동을 결정하는 노드"""
    # LLM 호출 로직
    response = llm.invoke(state["messages"])
    return {"messages": state["messages"] + [response]}

def execute_tool(state: AgentState) -> AgentState:
    """도구를 실행하는 노드"""
    # Tool 실행 로직
    result = tool_executor.invoke(state["messages"][-1])
    return {"messages": state["messages"] + [result]}

def should_continue(state: AgentState) -> str:
    """계속 실행할지 종료할지 결정하는 조건 엣지"""
    last_message = state["messages"][-1]
    if "tool_calls" in last_message:
        return "execute_tool"   # 도구 실행 필요
    return END                  # 완료

# 그래프 구성
workflow = StateGraph(AgentState)
workflow.add_node("call_llm", call_llm)
workflow.add_node("execute_tool", execute_tool)

workflow.set_entry_point("call_llm")
workflow.add_conditional_edges("call_llm", should_continue)
workflow.add_edge("execute_tool", "call_llm")  # 도구 실행 후 다시 LLM으로

agent = workflow.compile()
```

---

## 📌 LangChain vs LangGraph

| 구분 | LangChain | LangGraph |
|------|-----------|-----------|
| 구조 | 선형 체인 | 그래프 (노드 + 엣지) |
| 흐름 제어 | 순차적 | 조건부 분기, 루프 가능 |
| 상태 관리 | 제한적 | 명시적 State 관리 |
| 복잡한 에이전트 | 어려움 | 적합 |
| 학습 난이도 | 쉬움 | 보통 |
| 적합한 경우 | 단순 RAG, 체인 | 복잡한 멀티스텝 에이전트 |

> 채용공고 분석 에이전트처럼 **수집 → 분석 → 저장** 의 복잡한 흐름이라면 **LangGraph가 적합!**

---

## 📌 실전 적용 — 채용공고 분석 에이전트 설계

```
[에이전트 목표]
채용공고를 수집하고 분석해서 DB에 저장한다.

[플로우]
START
  ↓
① 채용공고 수집 노드 (Playwright로 크롤링)
  ↓
② 분석 노드 (LLM으로 요구사항 추출)
  ↓
③ 조건 분기
  - 분석 성공 → ④ DB 저장 노드
  - 분석 실패 → ② 재시도 (최대 3회)
  ↓
④ DB 저장 노드 (PostgreSQL)
  ↓
END

[사용 도구]
- scrape_job_posting: Playwright로 채용공고 크롤링
- analyze_with_llm: Groq API로 요구사항 분석
- save_to_db: PostgreSQL에 결과 저장
```

---

## 💭 오늘 느낀 점

- 에이전트는 "LLM을 여러 번 호출하는 것"이 아니라 LLM이 스스로 판단하고 도구를 선택한다는 점에서 패러다임이 다르다
- ReAct 패턴의 Thought → Action → Observe 루프가 인간의 문제 해결 방식과 매우 유사하다는 게 흥미롭다
- 지금 만들고 있는 채용공고 분석 프로젝트가 딱 LangGraph 에이전트 구조라는 걸 이제 명확히 이해했다
- LangChain은 단순 RAG, LangGraph는 복잡한 멀티스텝 에이전트 — 용도에 맞게 선택해야겠다

---

## 📚 참고

- [LangGraph 공식 문서](https://langchain-ai.github.io/langgraph/)
- [LangChain 공식 문서](https://python.langchain.com/docs/get_started/introduction)
- [ReAct 논문](https://arxiv.org/abs/2210.03629)
