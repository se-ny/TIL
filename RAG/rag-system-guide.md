# FastAPI와 LangChain을 활용한 문맥 유지형 RAG 시스템 구축

## 1. 프로젝트 개요
보유한 PDF 문서를 기반으로 대화 맥락을 유지하며 답변하는 로컬 RAG 시스템입니다.

## 2. 프로젝트 구조 예
2-1. System Architecture: 전체적인 작동 원리 (이용자 입력 → 벡터 검색 → LLM 응답)

2-2. Tech Stack:

- LLM: Gemma (via Ollama)

- Framework: LangChain, FastAPI

- Vector DB: Chroma DB

- Embedding: nomic-embed-text

2-3. Data Ingestion: PDF 문서를 쪼개고(Chunking) 벡터화하여 저장하는 과정

2-4. Inference & Performance: Gemma 모델별 응답 속도 테스트 결과

2-5. Advanced Features:

- Context Retention: 대화 이력 요약 및 DB 저장 방식

- Web Integration: FastAPI 세션 관리를 통한 사용자별 독립적 대화 유지

+ 추가할 내용
+ 설치 및 실행 명령어, 성능 테스트 결과 데이터, 코드 스니펫
