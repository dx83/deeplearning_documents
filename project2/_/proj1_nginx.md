# nginx 설명

이 프로젝트의 nginx는 외부 공개용 웹서버라기보다, Docker 안에 여러 개로 나뉜 프론트엔드/백엔드 서비스를 하나의 진입점으로 묶어 주는 리버스 프록시 역할을 한다.

현재 설정 기준으로 사용자가 접속하는 주소는 `http://localhost:8000`이고, 이 요청을 nginx가 내부 컨테이너인 `pill_chatbot_front`, `frontend_top3`, `backend`, `chat_backend`로 나누어 전달한다.

## nginx가 하는 일

nginx는 클라이언트 요청을 받아서 실제 처리할 서비스로 넘겨 주는 중간 서버다.

이 프로젝트에서는 크게 네 가지 역할을 한다.

1. 여러 서비스를 하나의 포트로 묶는다.
2. URL 경로에 따라 요청을 다른 컨테이너로 보낸다.
3. 브라우저 CORS 문제를 줄인다.
4. 정적 파일과 프론트엔드 빌드 결과물을 연결한다.

## 현재 구조

`docker/docker-compose.yaml`에서 nginx는 다음 포트로 열린다.

```yaml
nginx:
  ports:
    - "8000:80"
```

즉 사용자는 브라우저에서 `http://localhost:8000`으로 접속하고, Docker 내부에서는 nginx가 각 서비스 이름으로 통신한다.

```text
브라우저
  -> localhost:8000
      -> nginx
          -> pill_chatbot_front:80
          -> frontend_top3:80
          -> backend:8000
          -> chat_backend:8000
```

Docker 내부 컨테이너끼리는 `backend`, `chat_backend`, `argos`, `hermes`, `chromadb` 같은 서비스 이름으로 접근할 수 있다. 반면 브라우저는 Docker 내부 서비스 이름을 알 수 없기 때문에, nginx가 중간에서 외부 주소와 내부 서비스 주소를 연결한다.

## 현재 nginx 라우팅

현재 `docker/nginx/nginx.conf` 기준 주요 라우팅은 다음과 같다.

| 외부 경로 | nginx가 전달하는 대상 | 의미 |
| --- | --- | --- |
| `/` | `pill_chatbot_front:80` | 기본 화면 |
| `/top3`, `/top3/` | `frontend_top3:80/pill/top3` | top3 프론트 화면 |
| `/top3-assets/` | `frontend_top3:80/assets/` | top3 프론트 정적 파일 |
| `/pill/chat` | `backend:8000/pill/chat` | 이미지/메시지 기반 채팅 API |
| `/api/pill/demo` | `backend:8000/pill/demo` | 알약 이미지 분석 API |
| `/api/pill/top3` | `backend:8000/pill/top3` | top3 후보 분석 API |
| `/api/` | `backend:8000/` | 일반 backend API |
| `/classify` | `chat_backend:8000/classify` | 텍스트 챗봇 분류 API |
| `/clarify` | `chat_backend:8000/clarify` | 추가 질문 API |
| `/confirm` | `chat_backend:8000/confirm` | 후보 확인 API |
| `/feedback` | `chat_backend:8000/feedback` | 피드백 저장 API |
| `/stats` | `chat_backend:8000/stats` | 통계 API |
| `/pill/` | `chat_backend:8000` | 알약 상세 조회 API |
| `/chat/docs` | `chat_backend:8000/docs` | chat_backend Swagger 문서 |
| `/docs` | `backend:8000/docs` | backend Swagger 문서 |
| `/images/` | `/usr/share/nginx/html/images/` | 이미지 정적 파일 |

## 프론트엔드와 nginx의 관계

프론트엔드 코드는 현재 상대 경로로 API를 호출한다.

`frontend/src/services/api.js`:

```js
const API_BASE = "/api";
```

그래서 브라우저는 다음처럼 요청한다.

```text
http://localhost:8000/api/pill/demo
http://localhost:8000/api/pill/top3
http://localhost:8000/api/pill/chat
```

이 요청을 nginx가 `backend:8000`으로 넘긴다.

`pill_chatbot_front/src/api/chatbotApi.js`도 base URL이 빈 문자열이다.

```js
const CHATBOT_BASE_URL = "";
```

그래서 브라우저는 현재 접속한 서버 기준으로 요청한다.

```text
http://localhost:8000/classify
http://localhost:8000/clarify
http://localhost:8000/confirm
http://localhost:8000/feedback
```

이 요청은 nginx가 `chat_backend:8000`으로 넘긴다.

`pill_chatbot_front/src/api/imageApi.js`의 `/pill/chat` 요청은 nginx가 `backend:8000/pill/chat`으로 넘긴다.

## nginx를 쓰는 장점

nginx를 쓰면 브라우저 입장에서는 서버가 하나만 있는 것처럼 보인다.

```text
브라우저 -> http://localhost:8000
```

실제로는 내부에서 여러 서비스로 나뉜다.

```text
/              -> pill_chatbot_front
/top3          -> frontend
/api/pill/demo -> backend
/classify      -> chat_backend
```

이 구조의 장점은 다음과 같다.

| 장점 | 설명 |
| --- | --- |
| 포트가 단순해짐 | 사용자는 `localhost:8000` 하나만 알면 됨 |
| CORS 문제가 줄어듦 | 프론트와 API가 같은 origin처럼 보임 |
| 프론트 코드가 단순해짐 | API 주소를 `http://localhost:8004`처럼 직접 쓰지 않아도 됨 |
| Docker 서비스명 사용 가능 | nginx는 Docker 내부에서 `backend:8000` 같은 이름으로 접근 가능 |
| 라우팅을 한 곳에서 관리 | 어떤 URL이 어떤 서비스로 가는지 nginx 설정에 모임 |

## nginx 없이도 가능한가

가능하다. 외부 서비스가 아니라 Docker 기반 로컬 실행만 한다면 nginx는 필수는 아니다.

다만 nginx를 없애면 브라우저가 각 백엔드 포트로 직접 요청해야 한다.

예시:

```text
브라우저
  -> localhost:5173  frontend
  -> localhost:8000  backend
  -> localhost:8004  chat_backend

backend
  -> argos:8000
  -> hermes:8000

chat_backend
  -> chromadb:8000
```

이 경우 필요한 변경은 다음과 같다.

1. `backend` 포트를 외부로 연다.
2. `chat_backend` 포트를 외부로 연다.
3. 프론트 API base URL을 절대 주소로 바꾼다.
4. 백엔드 CORS 설정에 프론트 주소를 허용한다.

예를 들면 프론트 API 주소는 이렇게 바뀐다.

```js
const API_BASE = "http://localhost:8000";
const CHATBOT_BASE_URL = "http://localhost:8004";
const IMAGE_BASE_URL = "http://localhost:8000";
```

단, 이 방식은 프론트가 어느 백엔드로 요청해야 하는지 직접 알아야 한다. 지금처럼 `/classify`는 `chat_backend`, `/pill/chat`은 `backend`로 나뉘는 구조에서는 프론트 코드나 Vite proxy 설정이 복잡해질 수 있다.

## nginx 없이 Docker 내부 통신은 가능한가

백엔드끼리의 통신은 nginx 없이 가능하다.

예를 들어 현재 `backend`는 Docker 내부에서 다음 주소로 다른 AI 서버를 호출한다.

```text
backend -> http://argos:8000
backend -> http://hermes:8000
```

`chat_backend`도 Docker 내부에서 ChromaDB를 호출한다.

```text
chat_backend -> chromadb:8000
```

이런 서버 간 통신은 브라우저를 거치지 않으므로 nginx가 필요 없다.

문제는 프론트엔드다. React/Vite 앱의 `fetch()`는 컨테이너 안이 아니라 사용자의 브라우저에서 실행된다. 브라우저는 `backend:8000` 같은 Docker 내부 이름을 해석하지 못한다. 그래서 nginx를 쓰지 않을 경우 `localhost:포트`로 백엔드들을 노출해야 한다.

## 현재 프로젝트에서 nginx를 유지하는 편이 좋은 경우

다음 조건이면 nginx를 유지하는 편이 편하다.

| 조건 | 이유 |
| --- | --- |
| 프론트 2개와 백엔드 2개를 동시에 띄움 | 하나의 진입점으로 묶는 게 단순함 |
| 브라우저에서 같은 주소로 모든 기능을 쓰고 싶음 | `localhost:8000` 하나로 접근 가능 |
| CORS 설정을 최소화하고 싶음 | 같은 origin처럼 보이게 만들 수 있음 |
| `/classify`, `/pill/chat`, `/api/pill/top3`처럼 경로별 대상이 다름 | nginx가 분기 처리 |
| 팀원이 Docker compose만 실행해서 테스트해야 함 | 접속 주소를 하나로 안내 가능 |

## nginx를 제거해도 되는 경우

다음 조건이면 nginx를 제거해도 된다.

| 조건 | 이유 |
| --- | --- |
| 프론트를 하나만 사용함 | 라우팅 분기가 단순해짐 |
| 각 서비스 포트를 직접 알고 써도 됨 | `localhost:5173`, `localhost:8000`, `localhost:8004` 방식 가능 |
| CORS 설정을 직접 관리할 수 있음 | 백엔드에서 프론트 주소를 허용하면 됨 |
| 개발자가 각 서비스를 개별 실행함 | Vite dev proxy나 절대 URL로 충분함 |

## 결론

nginx는 이 프로젝트에서 필수 비즈니스 로직은 아니다. 알약 분석, 챗봇 처리, AI 추론은 각각 `backend`, `chat_backend`, `argos`, `hermes`, `chromadb`가 담당한다.

하지만 현재 프론트엔드가 상대 경로 API 호출을 사용하고 있고, 여러 서비스를 하나의 주소로 묶고 있기 때문에 nginx가 빠지면 프론트 API 주소, 포트 노출, CORS 설정을 다시 정리해야 한다.

현재 구조를 그대로 Docker에 올려서 팀원이 쉽게 실행하게 하려면 nginx 유지가 편하다. 구조를 단순화하고 각 서비스 포트를 직접 호출해도 괜찮다면 nginx 없이도 연동할 수 있다.

