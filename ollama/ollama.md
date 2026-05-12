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
  > 한국의 수도는 어디인가? 영어로 답해주세요.
    
## 느낀 점
세상에 사람에게 도움이 되는 LLM이 많구나라고 다시한번 느꼈다.

## 참고 링크
- https://ollama.com
