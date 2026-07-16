# 1장 CNN + Transformer를 이용한 알약 각인 인식
## 이전 CRNN의 문제점
각인이 2줄인 경우
"C:\Workspace\deeplearning\deeplearning_documents\project2\kst\ocr_0_bg.png"
"C:\Workspace\deeplearning\deeplearning_documents\project2\kst\ocr_1_bg.png"
각인이 알약에 꽉차게 있는 경우
"C:\Workspace\deeplearning\deeplearning_documents\project2\kst\ocr_2_bg.png"
숫자와 알파벳을 구분 못하는 경우
"C:\Workspace\deeplearning\deeplearning_documents\project2\kst\ocr_3_bg.png"

# 2장 CNN + Transformer를 이용한 알약 각인 인식
## CRNN vs CNN + Transformer
| | CRNN | CNN + Transformer |
|---|---|---|
| 주요 강점 | 규칙적인 한 줄 텍스트를 효율적으로 인식 | 각인 영역 전체의 전역 문맥 파악 |
| 공간 정보 처리 | 일반적으로 특징 맵을 1차원 시퀀스로 변환 | 2D 위치 인코딩으로 행과 열 정보 유지 |
| 어려운 배치 | 문자 간격이 넓거나 정렬이 불규칙하면 대응이 제한적 | 어텐션으로 멀리 떨어지거나 불규칙한 위치의 문자를 연결 |
| 연산량 | 비교적 적음 | 시각 토큰 수가 증가할수록 연산량 증가 |

**핵심 메시지:** 실제 알약에서 나타나는 불규칙한 각인 배치를 더 잘 찾아내기 위해 더 높은 연산 비용을 감수합니다.

# 3장 CNN + Transformer를 이용한 알약 각인 인식
## 순차적 읽기에서 공간적 이해로
- 왼쪽: `CNN 특징 → 왼쪽에서 오른쪽으로 이어지는 시퀀스 → 순환 신경망 인식`으로 구성된 간단한 CRNN 흐름도
    - CNN이 추출한 특징을 가로 방향의 시퀀스로 변환하여, 글자를 정해진 순서에 따라 차례대로 인식합니다.
- 오른쪽: `CNN 특징 → 2D 위치 정보 → 전역 어텐션 → 각인 디코딩`으로 구성된 프로젝트 모델 흐름도
    - 2차원 위치 정보를 유지한 채 전체 영역의 관계를 동시에 분석하여, 다양한 방향과 배열의 각인을 효과적으로 인식합니다.
