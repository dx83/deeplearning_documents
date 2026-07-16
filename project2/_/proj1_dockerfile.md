frontend/Dockerfile  
  - React/Vite 프론트 빌드  
  - nginx 같은 정적 서버로 화면 제공  

backend/Dockerfile  
  - Python/FastAPI 백엔드 환경 구성  
  - requirements.txt 설치  
  - uvicorn으로 FastAPI 서버 실행  

chat_backend/Dockerfile  
  - 챗봇 FastAPI 서버 환경 구성  
  - requirements.txt 설치  
  - uvicorn으로 FastAPI 서버 실행  

ai-server/argos/Dockerfile  
  - 이미지 분석 AI 서버 환경 구성  
  - 모델 추론에 필요한 패키지 설치  
  - uvicorn으로 FastAPI 서버 실행  

ai-server/hermes/Dockerfile  
  - 챗봇/응답 생성 AI 서버 환경 구성  
  - uvicorn으로 FastAPI 서버 실행  
