# 폴더/서비스 간 통신 관계

현재 저장소에서 확인한 프론트엔드/백엔드/API 서비스 간 통신 관계만 정리했습니다.

## 확인된 통신 관계

| 호출하는 쪽 | 받는 쪽 | 경로/방식 | 근거 |
| --- | --- | --- | --- |
| `docker/nginx` | `pill_chatbot_front` | `/` 전체 기본 라우팅 | `docker/nginx/nginx.conf`의 `location /` |
| `docker/nginx` | `frontend` (`frontend_top3`) | `/top3`, `/top3/`, `/top3-assets/` | `docker/nginx/nginx.conf` |
| `docker/nginx` | `backend` | `/api/pill/top3`, `/api/pill/demo`, `/api/`, `/docs`, `/redoc`, `/openapi.json`, `/pill/chat` | `docker/nginx/nginx.conf` |
| `docker/nginx` | `chat_backend` | `/classify`, `/clarify`, `/confirm`, `/feedback`, `/stats`, `/pill/`, `/api/classify`, `/api/clarify`, `/api/confirm`, `/api/feedback`, `/api/stats`, `/api/health`, `/api/pill/`, `/chat/docs`, `/chat/redoc`, `/chat/openapi.json` | `docker/nginx/nginx.conf` |
| `frontend` | `backend` | 브라우저에서 `/api/pill/demo`, `/api/pill/top3`, `/api/pill/chat` 호출 후 nginx가 `backend`로 프록시 | `frontend/src/services/api.js`, `docker/nginx/nginx.conf` |
| `pill_chatbot_front` | `chat_backend` | 브라우저에서 `/classify`, `/clarify`, `/confirm`, `/feedback`, `/pill/{id}` 호출 후 nginx가 `chat_backend`로 프록시 | `pill_chatbot_front/src/api/chatbotApi.js`, `docker/nginx/nginx.conf` |
| `pill_chatbot_front` | `backend` | 브라우저에서 `/pill/chat` 호출 후 nginx가 `backend`로 프록시 | `pill_chatbot_front/src/api/imageApi.js`, `docker/nginx/nginx.conf` |
| `backend` | `ai-server/argos` | `http://argos:8000/v1/pill/...` 이미지 분석 API 호출 | `backend/app/core/config.py`, `backend/app/services/argos.py`, `backend/app/services/pill_analysis.py` |
| `backend` | `ai-server/hermes` | `http://hermes:8000/v1/pill/chat` 챗봇 API 호출 | `backend/app/core/config.py`, `backend/app/services/hermes.py` |
| `chat_backend` | `chromadb` | `chromadb.HttpClient(host=CHROMADB_HOST, port=CHROMADB_PORT)`로 컬렉션 조회 | `chat_backend/classifier/stage1_text.py`, `docker/docker-compose.yaml` |

## docker-compose 기준 의존 관계

아래는 `docker/docker-compose.yaml`에 `depends_on` 또는 환경변수로 잡혀 있지만, 현재 코드 검색에서 직접 API 호출이 명확하게 확인되지 않은 관계입니다.

| 서비스 | 의존 대상 | 비고 |
| --- | --- | --- |
| `backend` | `mongodb` | `MONGODB_URL=mongodb://mongodb:27017` 환경변수와 `depends_on` 존재. 현재 코드에서 MongoDB 사용 코드는 확인되지 않음. |
| `backend` | `chromadb` | `CHROMADB_HOST`, `CHROMADB_PORT` 환경변수와 `depends_on` 존재. 현재 `backend` 코드에서 ChromaDB 클라이언트 사용은 확인되지 않음. |
| `nginx` | `pill_chatbot_front`, `frontend`, `backend`, `chat_backend` | 리버스 프록시 대상이며 healthcheck 이후 기동. |

## 통신 관계가 확인되지 않은 폴더

| 폴더 | 확인 내용 |
| --- | --- |
| `ai-server/tensor` | Dockerfile/requirements는 있으나 현재 compose 및 코드 검색에서 다른 폴더와의 호출 관계가 확인되지 않음. |
| `data` | 데이터 폴더로 보이며 서비스 호출 주체 아님. |
| `docs` | 문서 폴더로 보이며 서비스 호출 주체 아님. |
| `script` | 스크립트 폴더로 보이며 상시 통신 서비스 관계는 확인하지 않음. |
| `알약외양분류모델` | 모델/정리 자료 성격으로 보이며 서비스 호출 주체는 확인되지 않음. |

## 요약 흐름

```text
사용자 브라우저
  -> docker/nginx
      -> pill_chatbot_front
      -> frontend
      -> backend
          -> ai-server/argos
          -> ai-server/hermes
      -> chat_backend
          -> chromadb
```

