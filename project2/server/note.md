# Docker 프론트엔드/백엔드 연결 정리

## 프로젝트 구조

```text
project2/
├─ backend/
│  ├─ Dockerfile
│  ├─ requirements.txt
│  └─ app/
├─ frontend/
│  ├─ Dockerfile
│  ├─ package.json
│  └─ src/
└─ docker/
   └─ docker-compose.yaml
```

이 프로젝트는 `docker/docker-compose.yaml`에서 프론트엔드와 백엔드를 각각 별도 컨테이너로 실행한다.

## Docker 실행 전 확인

Docker Desktop을 실행한 뒤 아래 명령으로 Docker가 동작하는지 확인한다.

```powershell
docker version
```

## 백엔드를 Docker에 연결하는 방법

백엔드는 `backend/Dockerfile`로 이미지를 만든다.

핵심 흐름:

1. `python:3.12-slim` 이미지를 사용한다.
2. 컨테이너 작업 폴더를 `/app`으로 설정한다.
3. `requirements.txt`를 복사하고 Python 패키지를 설치한다.
4. `app/` 폴더를 컨테이너 안으로 복사한다.
5. `uvicorn`으로 FastAPI 서버를 `0.0.0.0:8000`에서 실행한다.

`docker-compose.yaml`에서는 백엔드를 이렇게 연결한다.

```yaml
backend:
  container_name: p2-backend
  build:
    context: ../backend
    dockerfile: Dockerfile
  ports:
    - "8000:8000"
```

`8000:8000`은 내 PC의 `8000` 포트를 컨테이너의 `8000` 포트와 연결한다.

## 프론트엔드를 Docker에 연결하는 방법

프론트엔드는 `frontend/Dockerfile`로 이미지를 만든다.

핵심 흐름:

1. `node:22-alpine` 이미지를 사용한다.
2. 컨테이너 작업 폴더를 `/app`으로 설정한다.
3. `package.json`, `package-lock.json`을 복사하고 `npm ci`로 패키지를 설치한다.
4. 프론트엔드 소스 전체를 복사한다.
5. `npm run build`로 빌드한다.
6. `npm run preview`로 Vite 결과물을 `0.0.0.0:80`에서 실행한다.

`docker-compose.yaml`에서는 프론트엔드를 이렇게 연결한다.

```yaml
frontend:
  container_name: p2-frontend
  build:
    context: ../frontend
    dockerfile: Dockerfile
  ports:
    - "5173:80"
```

`5173:80`은 내 PC의 `5173` 포트를 컨테이너의 `80` 포트와 연결한다.

## 전체 실행 방법

프로젝트 루트 폴더에서 실행한다.

```powershell
docker compose -f docker/docker-compose.yaml up --build
```

실행 후 접속 주소:

```text
프론트엔드: http://localhost:5173
백엔드:     http://localhost:8000
```

## 일부 서비스만 실행

백엔드만 실행:

```powershell
docker compose -f docker/docker-compose.yaml up --build backend
```

프론트엔드만 실행:

```powershell
docker compose -f docker/docker-compose.yaml up --build frontend
```

## 종료 방법

실행 중인 터미널에서 `Ctrl + C`로 중지하거나, 아래 명령으로 컨테이너를 내린다.

```powershell
docker compose -f docker/docker-compose.yaml down
```

## 정리

- 백엔드는 `backend/Dockerfile`로 Python/FastAPI 실행 환경을 만든다.
- 프론트엔드는 `frontend/Dockerfile`로 Node/Vite 실행 환경을 만든다.
- `docker/docker-compose.yaml`은 두 컨테이너를 한 번에 빌드하고 실행한다.
- 외부 접속은 `ports` 설정으로 내 PC 포트와 컨테이너 포트를 연결해서 처리한다.
