# TIL: LLM API Integration with Groq

> 날짜: 2026-06-16  
> 태그: `#AI` `#LLM` `#Groq` `#FastAPI` `#Python`

---

## 📌 Groq이란?

**Groq** 은 LLM 추론(inference)에 특화된 하드웨어(LPU)를 기반으로 한 AI API 서비스다.

```
OpenAI 대비 장점:
✅ 속도가 압도적으로 빠름 (LPU 기반)
✅ 무료 티어 제공 (분당 요청 제한 있음)
✅ OpenAI 호환 API → 코드 거의 그대로 사용 가능

단점:
❌ 모델 선택지가 OpenAI보다 적음
❌ 무료 티어는 Rate Limit 존재
```

---

## 📌 기본 설정

```bash
# 설치
pip install groq python-dotenv
```

```python
# .env
GROQ_API_KEY=gsk_xxxxxxxxxxxxxxxxxxxx
```

```python
# config.py
from dotenv import load_dotenv
import os

load_dotenv()

GROQ_API_KEY = os.getenv("GROQ_API_KEY")
GROQ_MODEL = "llama-3.3-70b-versatile"  # 기본 모델
```

---

## 📌 주요 모델 비교

| 모델 | 특징 | 용도 |
|------|------|------|
| `llama-3.3-70b-versatile` | 고성능, 범용 | 복잡한 추론, RAG |
| `llama-3.1-8b-instant` | 빠름, 경량 | 간단한 QA, 실시간 응답 |
| `mixtral-8x7b-32768` | 긴 컨텍스트 (32K) | 긴 문서 처리 |
| `gemma2-9b-it` | Google 모델 | 일반 대화 |

---

## 📌 기본 API 호출

```python
from groq import Groq
import os

client = Groq(api_key=os.getenv("GROQ_API_KEY"))

# 단순 질문
def simple_chat(message: str) -> str:
    response = client.chat.completions.create(
        model="llama-3.3-70b-versatile",
        messages=[
            {"role": "user", "content": message}
        ],
        temperature=0.7,
        max_tokens=1024,
    )
    return response.choices[0].message.content

result = simple_chat("FastAPI와 Flask의 차이점을 3줄로 설명해줘")
print(result)
```

---

## 📌 System Prompt 활용

```python
def chat_with_system(
    user_message: str,
    system_prompt: str,
    temperature: float = 0.7
) -> str:
    response = client.chat.completions.create(
        model="llama-3.3-70b-versatile",
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_message},
        ],
        temperature=temperature,
        max_tokens=1024,
    )
    return response.choices[0].message.content


# 활용 예시
system = """너는 친절한 AI 취업 코치야.
주니어 개발자의 포트폴리오와 기술 스택을 분석해서
취업에 도움이 되는 구체적인 조언을 해줘."""

answer = chat_with_system(
    "저는 FastAPI, PostgreSQL, RAG 프로젝트를 만들었어요. AI 엔지니어로 취업하려면 뭘 더 준비해야 할까요?",
    system_prompt=system
)
print(answer)
```

---

## 📌 멀티턴 대화 (대화 히스토리 유지)

LLM은 상태가 없다(Stateless) → 이전 대화를 직접 전달해야 한다.

```python
from typing import List, Dict

class ChatSession:
    def __init__(self, system_prompt: str = ""):
        self.history: List[Dict] = []
        if system_prompt:
            self.history.append({"role": "system", "content": system_prompt})

    def chat(self, user_message: str) -> str:
        # 사용자 메시지 추가
        self.history.append({"role": "user", "content": user_message})

        response = client.chat.completions.create(
            model="llama-3.3-70b-versatile",
            messages=self.history,
            temperature=0.7,
            max_tokens=1024,
        )

        assistant_message = response.choices[0].message.content

        # 어시스턴트 답변도 히스토리에 추가
        self.history.append({"role": "assistant", "content": assistant_message})

        return assistant_message


# 사용 예시
session = ChatSession(system_prompt="너는 친절한 파이썬 튜터야.")
print(session.chat("FastAPI가 뭐야?"))
print(session.chat("그럼 Flask랑 비교하면?"))  # 이전 대화 맥락 유지
print(session.chat("어떤 걸 배우면 좋을까?"))  # 계속 이어짐
```

---

## 📌 스트리밍 응답 (Streaming)

긴 답변을 한 번에 기다리지 않고 실시간으로 출력하는 방식.

```python
def stream_chat(message: str):
    stream = client.chat.completions.create(
        model="llama-3.3-70b-versatile",
        messages=[{"role": "user", "content": message}],
        stream=True,  # 스트리밍 활성화
    )

    for chunk in stream:
        content = chunk.choices[0].delta.content
        if content:
            print(content, end="", flush=True)  # 실시간 출력
    print()  # 줄바꿈


stream_chat("파이썬의 장점을 설명해줘")
# → 한 글자씩 실시간으로 출력됨
```

---

## 📌 FastAPI와 통합 — 스트리밍 엔드포인트

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from groq import Groq
import os

app = FastAPI()
client = Groq(api_key=os.getenv("GROQ_API_KEY"))

# 일반 응답
@app.post("/chat")
async def chat(message: str):
    response = client.chat.completions.create(
        model="llama-3.3-70b-versatile",
        messages=[{"role": "user", "content": message}],
        temperature=0.7,
        max_tokens=1024,
    )
    return {"answer": response.choices[0].message.content}


# 스트리밍 응답 (ChatGPT처럼 실시간 출력)
@app.post("/chat/stream")
async def chat_stream(message: str):
    def generate():
        stream = client.chat.completions.create(
            model="llama-3.3-70b-versatile",
            messages=[{"role": "user", "content": message}],
            stream=True,
        )
        for chunk in stream:
            content = chunk.choices[0].delta.content
            if content:
                yield content

    return StreamingResponse(generate(), media_type="text/plain")
```

---

## 📌 JSON 구조화 출력

LLM 답변을 파싱해서 구조화된 데이터로 받는 패턴.

```python
import json

def extract_structured_data(text: str) -> dict:
    prompt = f"""다음 텍스트에서 정보를 추출해서 JSON으로만 반환해줘.
다른 설명이나 마크다운 없이 순수 JSON만 반환해줘.

텍스트: "{text}"

출력 형식:
{{
  "name": "이름",
  "skills": ["스킬1", "스킬2"],
  "experience_years": 숫자
}}"""

    response = client.chat.completions.create(
        model="llama-3.3-70b-versatile",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.0,  # 일관된 JSON 출력을 위해 0으로 설정
        max_tokens=512,
    )

    raw = response.choices[0].message.content.strip()

    # JSON 파싱
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        # 마크다운 코드블록 제거 후 재시도
        clean = raw.replace("```json", "").replace("```", "").strip()
        return json.loads(clean)


result = extract_structured_data(
    "저는 3년차 개발자 김센아입니다. FastAPI, PostgreSQL, RAG를 다룰 수 있어요."
)
print(result)
# → {"name": "김센아", "skills": ["FastAPI", "PostgreSQL", "RAG"], "experience_years": 3}
```

---

## 📌 에러 핸들링

```python
from groq import RateLimitError, APIError
import time

def safe_chat(message: str, max_retries: int = 3) -> str:
    for attempt in range(max_retries):
        try:
            response = client.chat.completions.create(
                model="llama-3.3-70b-versatile",
                messages=[{"role": "user", "content": message}],
                max_tokens=1024,
            )
            return response.choices[0].message.content

        except RateLimitError:
            # Rate Limit 초과 → 잠시 대기 후 재시도
            wait_time = 2 ** attempt  # 지수 백오프: 1초, 2초, 4초
            print(f"Rate limit 초과. {wait_time}초 후 재시도...")
            time.sleep(wait_time)

        except APIError as e:
            print(f"API 오류: {e}")
            raise

    raise Exception("최대 재시도 횟수 초과")
```

---

## 📌 토큰 사용량 확인

```python
response = client.chat.completions.create(
    model="llama-3.3-70b-versatile",
    messages=[{"role": "user", "content": "안녕!"}],
    max_tokens=100,
)

usage = response.usage
print(f"입력 토큰: {usage.prompt_tokens}")
print(f"출력 토큰: {usage.completion_tokens}")
print(f"총 토큰:   {usage.total_tokens}")
```

---

## 💭 오늘 느낀 점

- Groq API가 OpenAI와 거의 동일한 인터페이스라서 코드 전환이 매우 쉽다는 걸 실감했다
- 멀티턴 대화에서 히스토리를 직접 관리해야 한다는 점 — LLM이 Stateless라는 걸 코드로 체감했다
- JSON 구조화 출력 시 `temperature=0.0` 으로 설정하는 것과 파싱 예외처리가 실전에서 꼭 필요하겠다
- 스트리밍 + FastAPI `StreamingResponse` 조합이 실제 챗봇 서비스 구현의 핵심 패턴인 것 같다

---

## 📚 참고

- [Groq 공식 문서](https://console.groq.com/docs/openai)
- [Groq Python SDK](https://github.com/groq/groq-python)
- [FastAPI StreamingResponse 문서](https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse)
