# router_kjs 챗봇 동작 분석

## 1. 결론

`router_kjs`의 챗봇은 LLM이 답변을 생성하는 방식이 아니다. 사용자의 문장에서 색상, 모양, 각인, 그림, 분할선 등의 조건을 규칙으로 추출한 뒤 ChromaDB의 알약 메타데이터를 조회하고 후보를 단계적으로 필터링하는 검색형 챗봇이다.

현재 핵심 흐름은 다음과 같다.

```text
POST /api/backend/kjs/classify
        │
        ▼
사용자 문장 규칙 기반 파싱
        │
        ├─ 색상
        ├─ 모양
        ├─ 영문·숫자 각인
        ├─ 그림 마크
        ├─ 분할선
        └─ 캡슐 여부
        │
        ▼
Stage 1: 각인·그림·분할선으로 ChromaDB 후보 조회
        │
        ▼
Stage 2: 모양 그룹으로 후보 필터링
        │
        ▼
Stage 3: Lab 색상 거리 계산 후 정렬
        │
        ▼
후보 수에 따라 결과 반환 또는 추가 질문
```

핵심 진입 함수는 `chatbot_core/chatbot/pipeline.py`의 `classify_pill()`이다.

## 2. API 구성

라우터 기본 주소는 다음과 같다.

```text
/api/backend/kjs
```

주요 API는 다음과 같다.

| API | 역할 |
|---|---|
| `POST /classify` | 최초 사용자 문장 분류 |
| `POST /clarify` | 분할선 또는 각인에 대한 추가 답변 처리 |
| `POST /confirm` | 추천한 알약이 맞는지 확인 |
| `POST /feedback` | 틀린 결과와 정답 기록 |
| `GET /pill/{pill_id}` | 알약 상세 정보 조회 |
| `GET /stats` | 누적 피드백 통계 조회 |

최초 요청 예시는 다음과 같다.

```json
{
  "text": "T가 새겨진 하얗고 동그란 약, 가운데 분할선 있어요"
}
```

`endpoints/classify.py`는 입력 문장을 `classify_pill()`에 전달하고 결과를 API 응답 모델로 변환한다.

## 3. 사용자 입력 파싱

`chatbot_core/parser/input_parser.py`의 `parse_input()`이 자연어 문장을 구조화한다.

예상 파싱 결과는 다음과 같다.

```json
{
  "color": "하양",
  "shape": "원형",
  "shape_group": "원형류",
  "text_marks": ["T"],
  "image_marks": [],
  "has_split_line": true,
  "is_capsule": false,
  "text_explicitly_none": false,
  "raw_text": "T가 새겨진 하얗고 동그란 약, 가운데 분할선 있어요"
}
```

추출 방법은 다음과 같다.

- 영문·숫자 각인: 정규식
- 색상·모양: 동의어 사전 substring 검색
- 한 글자 표현: Kiwi 형태소 분석
- 오타: RapidFuzz 문자열 유사도
- 분할선·캡슐: 키워드 및 주변 부정 표현 검사
- 그림 마크: Kiwi가 추출한 명사에서 일반 명사와 색상·모양 표현 제외

이 단계에서 이미 `동그란 → 원형`, `하얀 → 하양`과 같이 표준값으로 변환한다.

## 4. Stage 1: 각인·그림·분할선 검색

`chatbot_core/classifier/stage1_text.py`의 `classify_stage1()`이 ChromaDB의 `pill_data` 컬렉션을 조회한다.

검색 조건은 다음과 같다.

- 분할선 있음 또는 없음
- 글자 없음
- 앞면·뒷면 영문 또는 숫자 각인
- 앞면·뒷면 그림 마크
- 캡슐 여부

각인은 DB 값의 변형도 만들어 비교한다.

```text
DB 값: GX분할선623

비교 가능한 형태:
- GX분할선623
- GX
- 623
- GX623
```

반복된 각인도 축약한다.

```text
ITTITT → ITT
```

캡슐 데이터는 각인 부분 문자열 검색도 수행한다.

```text
사용자 입력: DK
DB 각인: KDAclang
결과: 부분 매칭
```

ChromaDB를 사용하지만 벡터 유사도 검색은 수행하지 않는다. `collection.query()`가 아니라 `collection.get(where=...)`와 Python 문자열 비교를 사용한다.

## 5. Stage 2: 모양 필터

`pipeline.py`는 Stage 1 후보를 사용자가 입력한 모양의 시각적 그룹으로 필터링한다.

정확한 모양 하나만 허용하는 것이 아니라 같은 그룹에 속한 모양을 모두 허용한다.

```text
사용자 표현: 동그란
표준 모양: 원형
그룹: 원형류
허용 후보: 원형, 구형, 도넛형 등
```

그룹 정보는 `classifier/stage2_shape.py`의 `get_group_members_by_standard()`에서 가져온다.

`resolve_shape_to_standard()` 함수는 파이프라인에 import되어 있지만 실제 호출되지 않는다. 모양 표준화는 앞 단계인 `input_parser.py`가 수행한다.

## 6. Stage 3: 색상 거리 계산 및 정렬

`pipeline.py`는 각 후보의 앞면·뒷면 색상과 사용자 색상의 Lab 거리를 계산한다.

```text
사용자 표준 색상 Lab 좌표
          ↕ ΔE76
DB 앞면·뒷면 색상 Lab 좌표
```

앞면과 뒷면 중 더 가까운 거리를 후보 점수로 사용한다.

```python
dist = min(dist_front, dist_back)
```

거리가 작을수록 후보 목록 위로 정렬된다. 거리가 `35`를 초과하면 후보에 `needs_confirmation=True`가 설정된다.

Stage 3는 후보를 제거하지 않고 순서만 정렬한다. 따라서 일반적으로 다음 관계가 성립한다.

```text
stage2_count == stage3_count
```

`classifier/stage3_color.py`의 `resolve_color_to_standard()`는 파이프라인에서 호출되지 않는다. 실제 파이프라인은 이 파일에 로드된 Lab 데이터와 `lab_distance()`만 사용한다.

## 7. 결과 및 추가 질문

`chatbot_core/api/responses.py`의 `build_response_from_result()`가 후보 수와 상태에 따라 응답 종류를 결정한다.

| 조건 | 응답 kind | 동작 |
|---|---|---|
| Stage 1 후보가 50개 초과이고 분할선 정보 없음 | `ask_split_line` | 분할선 여부 질문 |
| Stage 1 후보가 없고 각인 정보도 없음 | `ask_text_marks` | 글자나 그림 질문 |
| 최종 후보 0개 | `not_found` | 검색 실패 안내 |
| 후보 1개이고 색상 거리가 큼 | `confirm_single` | 사용자 확인 요청 |
| 후보 1개 | `single_result` | 단일 알약 반환 |
| 후보 여러 개 | `multiple` | 후보 목록 반환 |

추가 질문 응답에는 현재 파싱 정보가 `context`로 포함된다.

```json
{
  "kind": "ask_split_line",
  "question": "후보가 너무 많아요. 알약 가운데 분할선이 있나요?",
  "options": ["있음", "없음", "모름"],
  "context": {}
}
```

프런트엔드는 `context`를 보관했다가 `/clarify` 요청에 다시 전달한다. 서버가 별도의 대화 세션을 유지하는 구조가 아니라 클라이언트가 대화 상태를 들고 다니는 구조다.

## 8. 피드백 처리

사용자가 결과가 맞는지 확인하면 `chatbot_core/chatbot/feedback_logger.py`가 결과를 다음 파일에 JSON Lines 형식으로 기록한다.

```text
chatbot_core/feedback/log.jsonl
```

기록 내용에는 다음 정보가 들어간다.

- 최초 사용자 입력
- 예측 알약 ID와 이름
- 정답 여부
- 사용자가 제공한 정답 ID와 이름
- 파싱 결과

`training/incorporate_feedback.py`는 이 피드백을 SBERT 학습 쌍으로 변환할 수 있다. 다만 이는 수동 실행용 학습 스크립트이며 API 요청 과정에서 자동 재학습하지 않는다.

## 9. SBERT와 현재 챗봇의 관계

`parser/sbert_resolver.py`에는 텍스트를 768차원 임베딩으로 변환하고 표준 색상·모양과 코사인 유사도를 계산하는 구현이 있다.

하지만 `resolve_with_sbert()`는 실제 분류 파이프라인에서 호출되지 않는다. 현재 사용되는 흐름은 다음과 같다.

```text
사전 + Kiwi + RapidFuzz
        ↓
Stage 1 → Stage 2 → Stage 3
```

원래 의도한 것으로 추정되는 흐름은 다음과 같다.

```text
사전 + Kiwi + RapidFuzz
        │ 찾지 못함
        ▼
      SBERT
        │
        ▼
Stage 1 → Stage 2 → Stage 3
```

따라서 현재 챗봇은 SBERT 기반 의미 검색 챗봇이 아니라 규칙 기반 검색 챗봇에 가깝다.

## 10. 발견된 주요 문제

### 10.1 색상과 모양만으로 검색할 수 없음

Stage 1은 각인, 그림, 분할선 또는 `글자 없음` 정보가 모두 없으면 빈 후보를 반환한다.

```text
입력: 하얀 원형 약
결과: 색상·모양으로 전체 DB를 검색하지 않고 각인을 되물음
```

색상과 모양은 Stage 1에서 얻은 후보를 후처리할 뿐, 최초 후보를 생성하지 못한다.

### 10.2 ChromaDB 임베딩이 검색에 사용되지 않음

`db/load_to_chroma.py`는 알약 문서를 SBERT 임베딩과 함께 ChromaDB에 저장한다. 그러나 현재 검색 코드에는 `collection.query()`가 없고 메타데이터 필터와 문자열 비교만 존재한다.

따라서 저장된 알약 문서 임베딩은 현재 검색 경로에서 사용되지 않는다.

### 10.3 SBERT fallback 미연결

`sbert_resolver.py`와 `sbert_standard_cache.json`은 존재하지만 실제 파이프라인 호출처가 없다. 사전과 RapidFuzz가 처리하지 못한 표현을 SBERT가 보완하지 못한다.

### 10.4 일부 함수와 상수가 미사용 상태

현재 파이프라인에서 실질적으로 사용되지 않는 주요 요소는 다음과 같다.

- `stage2_shape.resolve_shape_to_standard()`
- `stage3_color.resolve_color_to_standard()`
- `stage3_color.find_neighbor_standards()`
- `pipeline.COLOR_DISTANCE_GOOD`
- `stage1_text._exact_match_text_marks()`

### 10.5 다중 후보의 색상 불일치 처리 부족

색상 거리가 큰 후보에는 `needs_confirmation=True`가 붙는다. 그러나 별도의 경고 응답은 최종 후보가 정확히 1개일 때만 발생한다. 후보가 여러 개면 색상 거리가 나쁜 후보도 그대로 목록에 포함된다.

### 10.6 전체 데이터 조회로 인한 성능 문제 가능성

분할선 조건이 없을 때 각인이나 그림을 검색하면 ChromaDB에서 전체 메타데이터를 가져와 Python 반복문으로 비교하는 경우가 있다. 데이터가 커질수록 요청 시간과 메모리 사용량이 증가할 수 있다.

### 10.7 ChromaDB에 대한 import 시점 강한 의존성

`stage1_text.py`는 모듈 import 시 바로 다음 작업을 수행한다.

```python
_client = chromadb.HttpClient(...)
_collection = _client.get_collection("pill_data")
```

ChromaDB 서버가 실행 중이지 않거나 `pill_data` 컬렉션이 없으면 FastAPI 애플리케이션 import 또는 시작 과정이 실패할 가능성이 있다.

### 10.8 Chroma ID와 품목일련번호 불일치

`load_to_chroma.py`는 ChromaDB ID를 실제 품목일련번호가 아닌 CSV 행 번호로 생성한다.

```python
ids=[str(j) for j in range(i, i + len(batch_df))]
```

피드백 화면에서는 사용자에게 품목일련번호를 입력하라고 하지만 사용자가 입력한 실제 품목일련번호는 ChromaDB의 행 번호 ID와 일치하지 않을 수 있다.

### 10.9 확인 API의 404 처리 문제

`endpoints/confirm.py`는 알약을 찾지 못했을 때 404 예외를 발생시키지만 바깥의 넓은 `except Exception`이 이를 다시 잡아 500 오류로 변환할 수 있다. `endpoints/pill.py`처럼 `except HTTPException: raise` 처리가 필요하다.

## 11. 최종 평가

`router_kjs`는 다음 구성요소를 갖춘 알약 검색 챗봇의 기본 골격이다.

- 자연어 조건 추출
- 각인과 분할선 중심 후보 검색
- 모양 그룹 필터
- Lab 색상 거리 정렬
- 추가 질문과 결과 확인
- 사용자 피드백 수집
- SBERT 학습 및 임베딩 실험 코드

다만 현재 상태에서는 각인 검색에 지나치게 의존하고 있으며 SBERT, ChromaDB 벡터 검색, 일부 Stage 함수가 실제 파이프라인에 연결되지 않았다. 따라서 완성된 의미 검색 챗봇이라기보다는 규칙 기반 검색 시스템에 SBERT 실험 코드가 추가된 중간 단계 구조로 판단된다.
