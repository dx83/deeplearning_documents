### 사전 필요한 폴더와 파일
`csv\pill_info_list.csv`  
`pill_images_data\learn\`  
`pill_images_data\test\`  
`saved\`

<br>

# 모델 학습
- cmd창
```
python pill_shape_classification.py
```
- 데이터 전처리
    - pill_shape_encoder.pkl
- 1차 학습
    - efficientnetb0_pill_shape_best_learning.keras
    - efficientnetb0_pill_shape_learning_history.csv
    - efficientnetb0_pill_shape_learning_history.pkl
- 파인 튜닝
    - efficientnetb0_pill_shape_best_tuning.keras
    - efficientnetb0_pill_shape_tuning_history.csv
    - efficientnetb0_pill_shape_tuning_history.pkl
- 최종 평가

<br>

# 클래스별 대표 임베딩 벡터 생성
- cmd창
```
python pill_shape_embedding_similarity.py
```
- 대표 임베딩 벡터와 유사도 통과 기준값 저장
    - pill_shape_centroids.pkl

<br>

# 예측
- 옵션에 사진 파일을 넣으면 1장 예측
```
python pill_shape_predict.py --image 사진파일
```

<br>

- `pill_images_data\test\` 시연 데이터 사용해서 예측
```
python pill_shape_model_evaluation.py
```
