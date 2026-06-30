---

## 📌 LangGraph 핵심 개념

### State (상태)

그래프 전체에서 공유되는 데이터.

```python
from typing import TypedDict, List

class AgentState(TypedDict):
    job_description: str
    quality_score: float
    analysis_result: dict
    retry_count: int
```

### Node (노드)

실제 작업을 수행하는 함수.

```python
def assess_quality(state: AgentState) -> AgentState:
    """채용공고 품질 평가 노드"""
    score = evaluate_job_posting(state["job_description"])
    return {**state, "quality_score": score}

def analyze_job(state: AgentState) -> AgentState:
    """LLM으로 채용공고 분석하는 노드"""
    result = llm_analyze(state["job_description"])
    return {**state, "analysis_result": result}
```

### Edge (엣지) & 조건부 엣지

```python
def should_analyze(state: AgentState) -> str:
    """품질 점수에 따라 다음 노드를 결정"""
    if state["quality_score"] >= 0.7:
        return "analyze_job"      # 품질 좋으면 분석 진행
    else:
        return "skip"             # 품질 낮으면 건너뛰기
```

---

## 📌 실전 비교 — 채용공고 분석 에이전트

### LangChain으로 구현 (한계 발생)

```python
# 단순 순차 체인 — 조건 분기 불가능
chain = quality_prompt | llm | analyze_prompt | llm | save_to_db
# "품질이 낮으면 분석 건너뛰기" 같은 로직을 넣기 어려움
```

### LangGraph로 구현 (자연스러움)

```python
from langgraph.graph import StateGraph, END

workflow = StateGraph(AgentState)

workflow.add_node("assess_quality", assess_quality)
workflow.add_node("analyze_job", analyze_job)
workflow.add_node("save_to_db", save_to_db)

workflow.set_entry_point("assess_quality")

# 조건부 분기 — 품질에 따라 다른 경로
workflow.add_conditional_edges(
    "assess_quality",
    should_analyze,
    {
        "analyze_job": "analyze_job",  # 품질 좋음 → 분석 진행
        "skip": END                     # 품질 낮음 → 바로 종료
    }
)

workflow.add_edge("analyze_job", "save_to_db")
workflow.add_edge("save_to_db", END)

agent = workflow.compile()

# 실행
result = agent.invoke({
    "job_description": "FastAPI 백엔드 개발자 모집...",
    "quality_score": 0.0,
    "analysis_result": {},
    "retry_count": 0
})
```

---

## 📌 언제 뭘 써야 할까?

| 상황 | 추천 |
|------|------|
| 단순 RAG (검색 → 답변) | **LangChain** — 충분하고 간단함 |
| 프롬프트 템플릿 관리 | **LangChain** — 표준 인터페이스 활용 |
| 조건 분기가 필요한 작업 | **LangGraph** — 명확한 분기 표현 |
| 재시도/루프가 필요한 작업 | **LangGraph** — 상태 기반 반복 |
| 여러 단계의 의사결정 에이전트 | **LangGraph** — ReAct 패턴 구현에 적합 |
| 진행상황 추적이 중요한 작업 | **LangGraph** — State로 명시적 관리 |

> 실무에서는 **LangChain의 컴포넌트(프롬프트, LLM 래퍼)를 LangGraph의 노드 안에서 함께 사용**하는 경우가 많다.

```python
# LangGraph 노드 안에서 LangChain 컴포넌트 사용
def analyze_job(state: AgentState) -> AgentState:
    chain = analysis_prompt | llm | output_parser  # LangChain 체인
    result = chain.invoke({"job": state["job_description"]})
    return {**state, "analysis_result": result}
```

---

## 📌 채용공고 에이전트 프로젝트와의 연결

---

## 💭 오늘 느낀 점

- LangChain은 "부품"을 제공하고, LangGraph는 그 부품들을 "어떤 순서로, 어떤 조건으로 조립할지"를 담당한다는 역할 분리가 명확해졌다
- 지금 프로젝트에서 한 `assess_post_quality` 조건부 분기가 정확히 LangGraph가 해결하려는 문제였다는 걸 이제 이론적으로 이해했다
- 두 프레임워크를 경쟁 관계가 아니라 LangChain(부품) + LangGraph(조립도)로 함께 쓰는 조합으로 이해하니 설계가 더 명확해졌다

---

## 📚 참고

- [LangChain 공식 문서](https://python.langchain.com/docs/get_started/introduction)
- [LangGraph 공식 문서](https://langchain-ai.github.io/langgraph/)
- [LangGraph Concepts 가이드](https://langchain-ai.github.io/langgraph/concepts/)
