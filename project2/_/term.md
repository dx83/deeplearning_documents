### requirements.txt
- 백엔드 Python 프로젝트에서 필요한 패키지 목록을 적어두는 파일입니다.

backend/Dockerfile
```
COPY requirements.txt .
RUN pip install --no-cache-dir --requirement requirements.txt
```
- Docker가 백엔드 컨테이너를 만들 때 requirements.txt를 보고 FastAPI, Uvicorn 같은 필요한 패키지를 설치

<br>

### Dockfile
- 각 서비스 컨테이너 이미지를 만드는 코드
- 이 폴더의 앱을 Docker 안에서 어떻게 설치하고 실행할지 적은 파일

```
Dockerfile     = 서비스 1개의 실행 환경 정의
docker-compose = 여러 서비스를 묶어서 실행
nginx          = 클라이언트 요청을 받아서 실제 처리할 서비스로 넘겨 주는 중간 서버
```

<br>

