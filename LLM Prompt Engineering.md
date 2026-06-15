# TIL: LLM Prompt Engineering

> 날짜: 2026-06-15  
> 태그: `#AI` `#LLM` `#PromptEngineering` `#RAG` `#Python`

---

## 📌 프롬프트 엔지니어링이란?

**프롬프트 엔지니어링(Prompt Engineering)** 은 LLM이 원하는 방향으로 답변하도록 **입력을 설계하는 기술**이다.

```
같은 LLM 모델이라도 프롬프트 설계에 따라 결과가 완전히 달라진다.

❌ 나쁜 프롬프트: "파이썬 코드 짜줘"
✅ 좋은 프롬프트: "FastAPI로 사용자 생성 POST /users 엔드포인트를 작성해줘.
                  Pydantic으로 요청 바디를 검증하고, 201 상태코드를 반환해야 해.
                  코드만 작성하고 설명은 생략해줘."
```

---

## 📌 메시지 역할 (Role)

LLM API는 보통 3가지 역할의 메시지를 사용한다.

```python
messages = [
    {
        "role": "system",    # LLM의 페르소나/행동 지침 설정
        "content": "너는 친절한 Python 백엔드 개발 전문가야. 항상 코드 예시를 포함해서 답변해줘."
    },
    {
        "role": "user",      # 사용자 입력
        "content": "FastAPI에서 JWT 인증을 어떻게 구현해?"
    },
    {
        "role": "assistant", # LLM의 이전 답변 (멀티턴 대화 시 사용)
        "content": "JWT 인증은 다음과 같이 구현할 수 있습니다..."
    }
]
```

| 역할 | 설명 | 특징 |
|------|------|------|
| `system` | LLM의 행동 지침 | 대화 전체에 영향, 가장 강한 지시 |
| `user` | 사용자 질문/요청 | 매 턴마다 변경 |
| `assistant` | LLM의 답변 | 멀티턴에서 이전 답변 기록용 |

---

## 📌 핵심 프롬프트 기법

### 1️⃣ Zero-shot Prompting

아무 예시 없이 바로 요청하는 방식.

```python
prompt = "다음 리뷰가 긍정인지 부정인지 판단해줘: '배송이 너무 느려요'"
# → "부정"
```

---

### 2️⃣ Few-shot Prompting

몇 가지 예시를 보여주고 패턴을 학습시키는 방식. 정확도가 크게 올라간다.

```python
prompt = """다음 리뷰가 긍정인지 부정인지 판단해줘.

리뷰: "정말 맛있어요!" → 긍정
리뷰: "배송이 너무 느려요" → 부정
리뷰: "가격 대비 품질이 좋네요" → 긍정

리뷰: "포장이 엉망이었어요" → """
# → "부정"
```

---

### 3️⃣ Chain-of-Thought (CoT) Prompting

**"단계적으로 생각해줘"** 를 명시해서 복잡한 추론 문제의 정확도를 높이는 기법.

```python
# ❌ Without CoT
prompt = "철수는 사과 5개를 갖고 있었는데 3개를 먹고 2개를 받았다. 몇 개?"
# → 간혹 틀림

# ✅ With CoT
prompt = """철수는 사과 5개를 갖고 있었는데 3개를 먹고 2개를 받았다. 몇 개?
단계적으로 생각해줘."""
# → "처음 5개 → 3개 먹어서 2개 → 2개 받아서 4개. 정답: 4개"
```

---

### 4️⃣ Role Prompting

LLM에게 특정 전문가 역할을 부여해서 해당 분야의 답변 품질을 높이는 기법.

```python
system_prompt = """너는 10년 경력의 시니어 백엔드 개발자야.
코드 리뷰 시 보안, 성능, 가독성 3가지 관점에서 피드백을 줘.
각 관점을 헤더로 구분해서 작성해줘."""
```

---

### 5️⃣ Output Format 지정

원하는 출력 형식을 명시하면 파싱이 쉬워진다.

```python
prompt = """다음 텍스트에서 이름, 나이, 직업을 추출해서 JSON으로 반환해줘.
다른 설명 없이 JSON만 반환해줘.

텍스트: "안녕하세요, 저는 28살 개발자 김철수입니다."

출력 형식:
{
  "name": "...",
  "age": ...,
  "job": "..."
}"""

# → {"name": "김철수", "age": 28, "job": "개발자"}
```

---

## 📌 RAG에서의 프롬프트 설계

RAG에서 프롬프트 설계는 품질에 직접적인 영향을 미친다.

```python
def build_rag_prompt(query: str, context_chunks: list[str]) -> str:
    context = "\n\n---\n\n".join(context_chunks)

    return f"""너는 주어진 문서를 기반으로 질문에 답변하는 AI 어시스턴트야.

반드시 아래 규칙을 따라줘:
1. 답변은 반드시 [참고 문서] 내용에 근거해야 해
2. 문서에 없는 내용은 "제공된 문서에서 찾을 수 없습니다"라고 답해줘
3. 답변은 한국어로 작성해줘
4. 핵심만 간결하게 3문장 이내로 답변해줘

[참고 문서]
{context}

[질문]
{query}

[답변]"""
```

---

## 📌 Groq API 실전 코드

```python
from groq import Groq

client = Groq(api_key="YOUR_GROQ_API_KEY")

def ask_llm(
    user_message: str,
    system_prompt: str = "너는 친절한 AI 어시스턴트야.",
    model: str = "llama-3.3-70b-versatile",
    temperature: float = 0.7,
) -> str:
    response = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_message},
        ],
        temperature=temperature,
        max_tokens=1024,
    )
    return response.choices[0].message.content


# Few-shot 예시
few_shot_system = """너는 감성 분석 전문가야. 리뷰를 보고 긍정/부정만 답해줘.

예시:
Q: "정말 좋아요!" → A: 긍정
Q: "별로예요" → A: 부정
Q: "그냥 그래요" → A: 부정"""

result = ask_llm("포장이 너무 예뻤어요!", system_prompt=few_shot_system)
print(result)  # → 긍정
```

---

## 📌 temperature & 파라미터 이해

```python
# temperature — 답변의 창의성/다양성 조절
temperature=0.0   # 항상 동일한 답변 → 코드 생성, 데이터 추출에 적합
temperature=0.7   # 적당히 다양한 답변 → 일반 QA, RAG에 적합
temperature=1.0   # 창의적이고 다양한 답변 → 글쓰기, 브레인스토밍에 적합

# max_tokens — 생성할 최대 토큰 수
max_tokens=256    # 짧은 답변
max_tokens=2048   # 긴 답변 (코드, 문서 등)
```

---

## 📌 프롬프트 설계 체크리스트

```
✅ system 프롬프트로 역할과 행동 지침을 명확히 설정했는가?
✅ 출력 형식(JSON, 마크다운 등)을 명시했는가?
✅ "문서에 없으면 모른다고 해줘" 등 Hallucination 방지 지침이 있는가?
✅ 답변 길이/언어/톤을 지정했는가?
✅ Few-shot 예시가 필요한 작업인가?
✅ 복잡한 추론이 필요하면 CoT를 유도했는가?
```

---

## 💭 오늘 느낀 점

- 프롬프트 엔지니어링은 단순한 "잘 물어보기"가 아니라 LLM의 동작을 제어하는 엔지니어링이라는 걸 실감했다
- RAG에서 프롬프트 설계가 검색 품질만큼이나 최종 답변 품질에 영향을 미친다는 게 중요한 포인트
- temperature를 0에 가깝게 설정하면 일관된 답변을 얻을 수 있다 — 데이터 추출 작업에 바로 써먹을 수 있겠다

---

## 📚 참고

- [OpenAI 프롬프트 엔지니어링 가이드](https://platform.openai.com/docs/guides/prompt-engineering)
- [Groq API 공식 문서](https://console.groq.com/docs/openai)
- [Chain-of-Thought Prompting 논문](https://arxiv.org/abs/2201.11903)
