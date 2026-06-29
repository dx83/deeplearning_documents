### requirements.txt
- 백엔드 Python 프로젝트에서 필요한 패키지 목록을 적어두는 파일입니다.

backend/Dockerfile
```
COPY requirements.txt .
RUN pip install --no-cache-dir --requirement requirements.txt
```
- Docker가 백엔드 컨테이너를 만들 때 requirements.txt를 보고 FastAPI, Uvicorn 같은 필요한 패키지를 설치

### Dockfile
