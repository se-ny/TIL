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

## 3. Gemma 모델별 RAG 성능 테스트 결과
모델명 (Model) | 파라미터 규모 | 응답 시간 (Response Time) | 비고
--------------------------------------------------------------------------
gemma4:e4b | High | 약 30초 | 코드 최적화 후 결과
--------------------------------------------------------------------------
gemma4:e2b | Medium | 약 15초
--------------------------------------------------------------------------
gemma2:9b | Medium | 약 15초
--------------------------------------------------------------------------
gemma2:2b | Small | 약 1초 | 가장 빠른 응답 속도
--------------------------------------------------------------------------

## 4. 개념/원리

- Vector DB
    - 벡터 저장: 임베딩 모델을 사용하여 텍스트를 벡터로 변환하여 저장
    - 의미 검색: 벡터 공간에서 크기/방향이 유사한 벡터를 다수개 찾아서 리턴
    이용자 텍스트 입력 → 임베딩 모델을 사용하여 벡터 변환 →벡터 DB에서 의미 검색 → 임베딩모델 → 텍스트 리턴
    - 검색된 벡터 → 텍스트로 변환 → 리턴
    

LangChain 모듈을 사용하여 pdf 문서를 VectorDB에 저장하기

- Embedding 모델을 사용하여 벡터화 하는 내용 포함
- Anaconda Prompt → conda env list → 가상환경 목록 확인(torch_env)
- conda activate torch_env
- D:/test/RAG ( or 현재 작업 폴더 에 PDF 파일 넣은 주소 ) → code . [ENTER]
- VS Code에서 파이썬 소스코드를 작성하여 pdf_embedding.py에 저장
- df_to_vectordb.py 파일이 저장된 곳에 pdf 파일도 저장
- VS Code 터미널에서 python pdf_to_vectordb.py [ENTER]
- 현재 디렉토리에 chroma_db 디렉토리가 생성되었는지 확인
    - 도메인문서파일(pdf)을 쪼개서 각각 벡터화하여 벡터DB에 저장하는 LangChain 코드
    - (밑에 코드 돌리기전에)
    pip install langchain 
    langchain-community
    langchain-ollama langchain-chroma pypdf
    
    from langchain_community.document_loaders import PyPDFLoader
    from langchain_text_splitters import RecursiveCharacterTextSplitter
    from langchain_ollama import OllamaEmbeddings
    from langchain_community.vectorstores import Chroma
    
    # 1. 문서 불러오기 (PDF 예시)
    
    loader = PyPDFLoader("도메인문서_파일.pdf")
    pages = loader.load()
    
    # 2. 문서 잘게 쪼개기 (Chunking)
    
    # AI가 한 번에 읽기 적당한 크기(500~1000자)로 나눕니다.
    
    text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=50
    )
    splits = text_splitter.split_documents(pages)
    
    # 3. 임베딩 모델 설정 (1단계에서 설치한 모델)
    
    embeddings = OllamaEmbeddings(model="nomic-embed-text")
    
    # 4. 벡터 DB 생성 및 저장
    
    # 현재 폴더의 './chroma_db' 경로에 데이터를 물리적으로 저장합니다.
    
    vectorstore = Chroma.from_documents(
    documents=splits,
    embedding=embeddings,
    persist_directory="./chroma_db"
    )
    
    print(f"문서가 {len(splits)}개의 조각으로 나누어져 DB에 저장되었습니다.")
    

이용자 입력 → AI 응답

이용자 입력 → EM → Vector DB → EM → 합친다 → LLM(Gemma4) → 완성된 텍스트

이용자 입력 → EM → Vector DB → EM → 합친다 → Ollama → 응답 텍스트
+ 추가할 내용
+ 설치 및 실행 명령어, 성능 테스트 결과 데이터, 코드 스니펫
