# TIL: RAG Principles & Pipeline

> 날짜: 2026-06-12  
> 태그: `#AI` `#RAG` `#LLM` `#Python` `#FastAPI`

---

## 📌 RAG란?

**RAG(Retrieval-Augmented Generation)** 는 LLM의 답변 생성에 **외부 문서 검색을 결합**한 기법이다.

### LLM만 쓸 때의 문제점

```
❌ Hallucination  — 모르는 내용을 그럴듯하게 지어냄
❌ 지식 한계      — 학습 데이터 이후의 정보는 모름
❌ 사내 문서 무지 — 우리 회사 내부 문서는 당연히 모름
```

### RAG가 해결하는 것

```
✅ 실제 문서에서 근거를 찾아 답변 → Hallucination 감소
✅ 최신 문서, 사내 문서도 검색 가능
✅ 출처 제공 가능 → 신뢰성 향상
```

---

## 📌 RAG 전체 파이프라인

RAG는 크게 **Indexing(사전 준비)** 과 **Querying(질의 처리)** 두 단계로 나뉜다.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Phase 1: Indexing] — 사전에 1번만 실행
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

원본 문서 (PDF, TXT 등)
    ↓
① Load       문서 로드
    ↓
② Chunk      청크(조각)로 분할
    ↓
③ Embed      각 청크를 벡터로 변환
    ↓
④ Store      벡터 DB에 저장


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Phase 2: Querying] — 사용자 질문마다 실행
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

사용자 질문
    ↓
⑤ Embed      질문을 벡터로 변환
    ↓
⑥ Retrieve   벡터 DB에서 유사한 청크 Top-K 검색
    ↓
⑦ Augment    질문 + 검색된 청크를 프롬프트로 조합
    ↓
⑧ Generate   LLM이 프롬프트 기반으로 답변 생성
    ↓
최종 답변
```

---

## 📌 각 단계 상세

### ① Load — 문서 로드

```python
# PyMuPDF로 PDF 텍스트 추출
import fitz  # pymupdf

def load_pdf(file_path: str) -> str:
    doc = fitz.open(file_path)
    text = ""
    for page in doc:
        text += page.get_text()
    return text
```

---

### ② Chunk — 청크 분할

문서를 통째로 임베딩하면 너무 크고 노이즈가 많다.  
적절한 크기로 잘라야 검색 정확도가 높아진다.

```python
def split_into_chunks(text: str, chunk_size: int = 500, overlap: int = 50) -> list[str]:
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start += chunk_size - overlap  # overlap으로 문맥 연속성 유지
    return chunks
```

> **overlap(겹침)이 왜 필요한가?**  
> 청크 경계에서 문장이 잘릴 수 있다. 앞뒤 청크가 조금씩 겹치면 문맥이 끊기지 않는다.

---

### ③④ Embed & Store — 임베딩 & 벡터 DB 저장

```python
from sentence_transformers import SentenceTransformer
import chromadb

model = SentenceTransformer('all-MiniLM-L6-v2')
client = chromadb.PersistentClient(path="./chroma_db")  # 디스크에 영구 저장
collection = client.get_or_create_collection("documents")

def index_chunks(chunks: list[str], doc_id: str):
    embeddings = model.encode(chunks).tolist()
    ids = [f"{doc_id}_{i}" for i in range(len(chunks))]
    collection.add(
        documents=chunks,
        embeddings=embeddings,
        ids=ids
    )
```

---

### ⑤⑥ Embed & Retrieve — 질문 임베딩 & 검색

```python
def retrieve(query: str, top_k: int = 3) -> list[str]:
    query_embedding = model.encode([query]).tolist()
    results = collection.query(
        query_embeddings=query_embedding,
        n_results=top_k
    )
    return results['documents'][0]  # 유사한 청크 top_k개 반환
```

---

### ⑦⑧ Augment & Generate — 프롬프트 조합 & LLM 답변 생성

```python
from groq import Groq

client = Groq(api_key="YOUR_API_KEY")

def generate_answer(query: str, context_chunks: list[str]) -> str:
    context = "\n\n".join(context_chunks)

    prompt = f"""다음 문서를 참고해서 질문에 답변해줘.
문서에 없는 내용은 "문서에서 찾을 수 없습니다"라고 답변해줘.

[참고 문서]
{context}

[질문]
{query}
"""
    response = client.chat.completions.create(
        model="llama-3.3-70b-versatile",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content
```

---

## 📌 전체 파이프라인 통합 코드

```python
# RAG 전체 흐름 통합
def rag_pipeline(file_path: str, query: str) -> str:
    # Phase 1: Indexing
    text = load_pdf(file_path)
    chunks = split_into_chunks(text, chunk_size=500, overlap=50)
    index_chunks(chunks, doc_id="my_doc")

    # Phase 2: Querying
    relevant_chunks = retrieve(query, top_k=3)
    answer = generate_answer(query, relevant_chunks)

    return answer

# 실행
answer = rag_pipeline("document.pdf", "이 문서의 핵심 내용이 뭐야?")
print(answer)
```

---

## 📌 RAG 품질을 높이는 핵심 포인트

```
청크 크기     chunk_size가 너무 크면 노이즈 증가, 너무 작으면 문맥 부족
              → 보통 300~800자, 도메인에 따라 조정

overlap       청크 경계의 문맥 단절 방지
              → 청크 크기의 10~20% 권장

임베딩 모델   한국어 문서라면 한국어 특화 모델 사용
              → bge-m3, ko-sroberta-multitask

Top-K         너무 적으면 정보 부족, 너무 많으면 LLM 컨텍스트 낭비
              → 보통 3~5개

프롬프트 설계 "문서에 없으면 모른다고 해줘" 명시 → Hallucination 방지
```

---

## 📌 DocuChat과의 연결

```
DocuChat에서 직접 구현한 것들:
✅ PDF 업로드 → PyMuPDF로 텍스트 추출       (① Load)
✅ LangChain RecursiveCharacterTextSplitter  (② Chunk)
✅ HuggingFace sentence-transformers         (③ Embed)
✅ ChromaDB에 벡터 저장                      (④ Store)
✅ 질문 임베딩 후 유사 청크 검색             (⑤⑥ Retrieve)
✅ Groq API (llama-3.3-70b) 답변 생성        (⑦⑧ Generate)
```

당시엔 각 부품을 따로 붙였지만, 이제 전체 흐름이 하나로 연결된다.

---

## 💭 오늘 느낀 점

- RAG는 "검색 + 생성"의 조합인데, 각 단계가 독립적으로 교체 가능한 파이프라인 구조라는 게 인상적이다
- DocuChat을 만들 때 LangChain이 이 파이프라인을 추상화해서 제공했다는 걸 이제야 제대로 이해했다
- 청크 크기와 overlap 튜닝이 RAG 품질에 직접적인 영향을 미친다는 것 — 다음 프로젝트에서 실험해보고 싶다

---

## 📚 참고

- [LangChain RAG 공식 문서](https://python.langchain.com/docs/use_cases/question_answering/)
- [ChromaDB 공식 문서](https://docs.trychroma.com/)
- [sentence-transformers 공식 문서](https://www.sbert.net/)
