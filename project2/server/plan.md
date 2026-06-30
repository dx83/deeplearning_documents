# Nginx 도입 계획

## 목표

현재 프로젝트의 Docker 기반 `frontend`와 `backend` 구성을 nginx 중심 구조로 정리한다.

- React/Vite 빌드 결과물을 nginx로 정적 서빙한다.
- `/api` 요청을 backend 컨테이너로 리버스 프록시한다.
- 외부에는 nginx만 노출하고, backend는 Docker 내부 네트워크에서만 접근하도록 정리한다.
- 개발/운영 구성을 분리할 수 있는 기반을 만든다.

## 현재 구조 요약

- `frontend`
  - Vite React 프로젝트
  - Dockerfile에서 `npm run build` 후 현재는 `npm run preview`로 80번 포트 서빙
- `backend`
  - Python 3.12 기반
  - `uvicorn app.main:app --host 0.0.0.0 --port 8000`
- `docker/docker-compose.yaml`
  - `backend`: 호스트 `8000:8000`
  - `frontend`: 호스트 `5173:80`

## 권장 구조

nginx를 프론트 정적 서버이자 API 게이트웨이로 사용한다.

```text
Client
  |
  v
nginx :80
  |-- /              -> frontend 정적 파일
  |-- /api/*         -> backend:8000
  |-- /health        -> nginx 또는 backend health check
```

권장 도입 방식은 `frontend` 이미지를 nginx 기반 멀티 스테이지 이미지로 바꾸는 것이다. 별도 `nginx` 서비스로 둘 수도 있지만, 현재 규모에서는 프론트 정적 파일과 프록시 설정을 한 컨테이너에 묶는 편이 단순하다.

## 작업 범위

### 1. nginx 설정 추가

추가 후보 파일:

- `frontend/nginx.conf`

포함할 설정:

- `root /usr/share/nginx/html;`
- SPA 라우팅 대응: `try_files $uri $uri/ /index.html;`
- API 프록시:

```nginx
location /api/ {
    proxy_pass http://backend:8000/;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

주의할 점:

- backend 라우트가 `/api` prefix를 기대하는지 확인해야 한다.
- backend가 `/api` 없는 경로를 기대한다면 위처럼 trailing slash를 둔다.
- backend가 `/api` 포함 경로를 기대한다면 `proxy_pass http://backend:8000;` 형태로 조정한다.

### 2. frontend Dockerfile을 nginx 기반으로 변경

현재:

- `node:22-alpine`에서 빌드
- `npm run preview`로 실행

변경 방향:

```dockerfile
FROM node:22-alpine AS build
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginx:1.27-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

효과:

- 운영 컨테이너에서 Node 런타임 제거
- Vite preview 의존 제거
- 정적 파일 서빙 성능 및 운영 안정성 개선

### 3. docker-compose 정리

현재는 backend와 frontend가 모두 호스트 포트를 노출한다.

변경 방향:

- `frontend` 또는 `nginx`만 호스트 포트 노출
- `backend`의 `ports` 제거 또는 개발용 override로 이동
- 서비스 간 통신은 Docker Compose 기본 네트워크의 서비스명 `backend` 사용

예상 변경:

```yaml
services:
  backend:
    container_name: p2-backend
    build:
      context: ../backend
      dockerfile: Dockerfile
    restart: unless-stopped
    expose:
      - "8000"

  frontend:
    container_name: p2-frontend
    build:
      context: ../frontend
      dockerfile: Dockerfile
    restart: unless-stopped
    ports:
      - "80:80"
    depends_on:
      - backend
```

개발 환경에서 backend 직접 접근이 필요하면 `docker-compose.override.yaml` 또는 별도 개발 compose 파일에서만 `8000:8000`을 노출한다.

### 4. API 경로 점검

확인 항목:

- frontend 코드에서 API base URL이 하드코딩되어 있는지 확인
- `http://localhost:8000` 같은 직접 호출이 있다면 `/api` 상대 경로로 변경
- backend 라우터가 `/api` prefix를 갖는지 확인
- CORS 설정이 남아 있다면 nginx 프록시 구조에서 필요한지 재검토

권장:

- 브라우저 기준 API 호출은 `/api/...` 형태로 통일
- 컨테이너 내부 주소인 `backend:8000`은 nginx 설정에만 둔다.

### 5. 헬스체크 추가

backend에 health endpoint가 있으면 nginx 또는 compose healthcheck에서 사용한다.

예:

- backend: `GET /health`
- nginx: `GET /health`를 backend로 프록시하거나 nginx 자체 `200 OK` 반환

도입 후보:

```nginx
location = /health {
    access_log off;
    return 200 "ok\n";
}
```

### 6. 검증 절차

로컬 검증:

```powershell
docker compose -f docker/docker-compose.yaml up --build
```

확인:

- `http://localhost/` 접속 시 프론트 화면 표시
- 브라우저 새로고침으로 SPA 라우팅 유지
- `http://localhost/api/...` 요청이 backend로 전달
- backend 포트 `8000`이 외부에 직접 열리지 않는지 확인
- 컨테이너 로그에 nginx 404, 502, upstream 오류가 없는지 확인

### 7. 배포 고려사항

운영 배포 전에 결정할 항목:

- TLS 종료를 nginx에서 할지, 상위 로드밸런서에서 할지
- 운영 도메인과 서버 포트 매핑
- gzip 또는 brotli 압축 적용 여부
- 정적 에셋 캐시 정책
- 업로드 용량 제한 필요 여부: `client_max_body_size`
- 프록시 타임아웃 정책
- access/error log 수집 방식

정적 에셋 캐시 예:

```nginx
location /assets/ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

## 단계별 일정

1. `frontend/nginx.conf` 추가
2. `frontend/Dockerfile`을 nginx 멀티 스테이지 구조로 변경
3. `docker/docker-compose.yaml`에서 외부 노출 포트를 nginx 중심으로 정리
4. frontend API 호출 경로를 `/api` 기준으로 점검 및 수정
5. Docker Compose로 전체 빌드 및 실행 검증
6. nginx 로그와 backend 로그를 확인하며 프록시 경로 보정
7. 운영 배포 옵션(TLS, 캐시, 압축, 업로드 제한)을 환경에 맞게 확정

## 리스크와 대응

- API prefix 불일치
  - backend 라우팅과 nginx `proxy_pass` trailing slash 동작을 함께 확인한다.
- SPA 라우팅 404
  - nginx `try_files $uri $uri/ /index.html;` 설정으로 대응한다.
- CORS 설정 혼선
  - 브라우저에서 같은 origin으로 `/api`를 호출하도록 정리하면 CORS 필요성이 줄어든다.
- backend 직접 접근 차단으로 인한 개발 불편
  - 개발용 compose override에서만 `8000:8000`을 열어 둔다.
- 파일 업로드 또는 긴 요청 실패
  - `client_max_body_size`, `proxy_read_timeout`, `proxy_send_timeout`을 요구사항에 맞게 조정한다.

## 완료 기준

- nginx가 프론트 정적 파일을 정상 서빙한다.
- `/api` 요청이 backend 컨테이너로 정상 프록시된다.
- 외부 공개 포트가 nginx 기준으로 정리된다.
- 새로고침과 직접 URL 진입에서도 SPA 라우팅이 깨지지 않는다.
- Docker Compose 한 번으로 frontend, backend, nginx 구성이 재현된다.
