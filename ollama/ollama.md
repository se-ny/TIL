docs: Ollama 공부 정리 Closes #1
# Ollama 사용법

## 한 줄 요약
Ollama는 AI 모델을 로컬에서 실행할 수 있는 툴

## 배운 내용
- LM Studio: GUI를 사용하여 로컬 시스템에서  LLM을 다운로드 및 시작/사용/종료 등 관리
    - 개인이 LLM을 다운로드하여 로컬 시스템에서 사용할 때 권장됨
    - 다양한 LLM을 다운로드하여 성능이나 한글 테스트하기에 적당함
- Ollama: 로컬 시스템에 LLM을 다운로드하고 시작/사용/종료 등 관리
    - Python 프로젝트에서 Ollama에 접속하여 LLM을 사용할 수 있으므로 기업용 Chatbot을 구축할 때 권장됨
    - 내부에서 http 서버를 내장
    - CMD/Pyhon에서도 사용 가능
    - https://ollama.com/ 에서 다운로드 및 설치
- Install
    - Setting > Model location : 주소
    - Setting > Context length : 우측 끝까지 드래그
    - Setting > Model Defaults > Default Context Length > Model maximum
- 이미 로컬시스템에 다운로드 된 Gemma 4를 사용할 것이므로 모델 다운로드과정 없이 등록만 하면 됨

## 설치 방법
- 해당 주소 알기
- Modelfile 파일 생성(확장자 없어야 함)
- Modelfile의 내용: FROM "./gemma-4-e4b-it-q4_k_m.gguf"
- 파일명은 모두 소문자이어야 함
- CMD 열고 ./ollama/models로 이동
- ollama create gemma4e4b -f Modelfile
  > gathering model components ..., writing mainfast, success

## 실행하는 방법
- ollama run gemma4e4b
  >ollama run gemma4 (ggf대신 ollama에서 자동으로 gemma4를 설치해서 cmd에서 열때 명령어 이거 적용)
      >한국의 수도는 어디인가? 영어로 답해주세요.
    
## Ollama 완전 정리
- 로컬 PC에서 AI모델을 쉽게 실행하게 해주는 도구
인터넷 없이, API 비용 없이, 내 컴퓨터에서 LLM을 돌리는것
    - 인터넷 없는 환경 → 오프라인에서 AI 사용
    - 비용 절감 → ChatGPT/Claude API 비용 0원
    - 개인정보 보호 → 데이터가 외부로 안 나감
    - 개발/테스트 → 로컬에서 AI 앱 개발할 때
- 무제한으로 쓰고 싶으면 Ollama+Open WebUI조합 최고
but 단점도 존재(속도 느림, 성능 못 따라감 등)

## 참고 링크
- https://ollama.com
