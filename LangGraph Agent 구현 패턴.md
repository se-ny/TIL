# LangGraph Agent 구현 패턴

## 왜 LangGraph인가?

LLM을 단순히 한 번 호출하는 것을 넘어서, **여러 단계를 순서대로 실행**하고 상태를 유지해야 할 때 LangGraph를 사용한다.

- 단순 LLM 호출: `llm.invoke(prompt)` → 1회성
- LangGraph: 노드(Node) + 엣지(Edge)로 워크플로우를 구성 → 상태 기반 멀티스텝

---

## 핵심 구성 요소

### 1. State (상태)

그래프 전체에서 공유되는 데이터 구조. `TypedDict`로 정의한다.

```python
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    job_posts: list[dict]       # 입력 데이터
    keywords_list: list[dict]   # 중간 결과
    stack_analysis: dict        # 중간 결과
    report_md: str              # 최종 결과
    errors: Annotated[list[str], operator.add]  # 누적 리스트
```

`Annotated[list, operator.add]`를 사용하면 각 노드에서 반환한 리스트가 **덮어쓰기가 아닌 누적**된다.

### 2. Node (노드)

상태를 받아서 일부를 수정해 반환하는 함수. 반환값은 업데이트할 키만 포함하면 된다.

```python
def node_extract(state: AgentState) -> dict:
    keywords_list = []
    for post in state["job_posts"]:
        result = extract_keywords.invoke({"job_text": post["description"]})
        keywords_list.append(result)
    return {"keywords_list": keywords_list}  # 변경된 키만 반환
```

async 노드도 가능하다:

```python
async def node_analyze_gap(state: AgentState) -> dict:
    result = await analyze_gap.ainvoke({...})
    return {"gap_analysis": result}
```

### 3. Graph 조립

```python
from langgraph.graph import StateGraph, END

def build_graph():
    graph = StateGraph(AgentState)

    # 노드 등록
    graph.add_node("extract", node_extract)
    graph.add_node("analyze_stack", node_analyze_stack)
    graph.add_node("report", node_report)

    # 시작점 지정
    graph.set_entry_point("extract")

    # 엣지 연결 (실행 순서)
    graph.add_edge("extract", "analyze_stack")
    graph.add_edge("analyze_stack", "report")
    graph.add_edge("report", END)

    return graph.compile()

job_agent = build_graph()
```

---

## 실행 방법

### 동기 실행
```python
final_state = job_agent.invoke({
    "job_posts": [...],
    "keywords_list": [],
    "stack_analysis": {},
    "report_md": "",
    "errors": [],
})
```

### 비동기 실행
```python
final_state = await job_agent.ainvoke({...})
```

`ainvoke`는 async 노드가 포함된 경우 반드시 사용해야 한다.

---

## 실제 적용 — job-agent 프로젝트

job-agent에서 채용공고 분석 파이프라인을 구성했다:

```
extract_keywords → analyze_stack → analyze_gap → report → END
```

| 노드 | 역할 |
|------|------|
| `extract_keywords` | 공고별 필수/우대 스킬, 경력, 직무 추출 |
| `analyze_stack` | 전체 공고 스택 빈도 분석 (Python 66.67%) |
| `analyze_gap` | DB 프로필과 비교해 보유/부족 스킬 도출 |
| `report` | 마크다운 리포트 생성 (매칭 점수 포함) |

각 노드는 Groq LLM을 호출하고 JSON 응답을 파싱해서 다음 노드로 전달한다.

---

## 핵심 포인트 정리

- State는 그래프 전체의 공유 메모리 역할을 한다
- 노드는 상태 전체를 받지만, 반환은 변경된 키만 하면 된다
- `Annotated[list, operator.add]`로 에러 같은 누적 데이터를 관리할 수 있다
- async 노드가 하나라도 있으면 `ainvoke`로 실행해야 한다
- `graph.compile()`로 컴파일 후 싱글턴으로 재사용하는 것이 효율적이다
