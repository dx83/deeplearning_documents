# Pill Shape Model Specification

## 1. 개요

이 문서는 알약 외형(shape) 분류 모델과 unknown 판정을 위한 centroid similarity index의 명세서이다.

- 모델 목적: 알약 이미지 1장을 입력받아 등록된 9개 외형 클래스 중 하나로 분류하고, 분류 확률과 임베딩 유사도 조건을 함께 사용해 known/unknown 여부를 판정한다.
- 최종 분류 클래스 수: 9개
- unknown 판정: softmax 분류 결과와 embedding centroid 유사도 결과가 모두 기준을 통과해야 known으로 처리한다.
- 현재 전처리 기준: 학습, centroid 생성, 예측 모두 `resize_with_pad` 기반으로 통일되었다.

## 2. 대상 파일

| 구분 | 파일 | 크기 | 설명 |
|---|---:|---:|---|
| 최종 모델 | `saved/efficientnetb0_pill_shape_best_tuning.keras` | 29,175,025 bytes | EfficientNetB0 fine-tuning 완료 모델 |
| 라벨 인코더 | `saved/pill_shape_encoder.pkl` | 352 bytes | 클래스명과 정수 label 매핑 |
| 유사도 인덱스 | `saved/pill_shape_centroids.pkl` | 47,116 bytes | 클래스별 embedding centroid, threshold, 통계 |
| 참고 문서 | `pill_shape_model_proto.md` | 8,429 bytes | 기존 모델 명세 초안 |

참고: 요청에는 `pill_shape_model_proto.md`가 `saved` 폴더에 있는 것으로 적혀 있었지만, 실제 파일은 `C:\Workspace\deeplearning\pill_shape\pill_shape_model_proto.md`에 있다.

## 3. 입력 명세

| 항목 | 값 |
|---|---|
| 입력 이미지 | JPG, JPEG, PNG, BMP, GIF |
| 입력 shape | `(None, 224, 224, 3)` |
| 이미지 크기 | 224 x 224 |
| 채널 | RGB 3채널 |
| 추론 전처리 | `tf.io.decode_image(channels=3)`, `tf.image.resize_with_pad(image, 224, 224)`, EfficientNet `preprocess_input` |

`resize_with_pad`는 원본 비율을 유지하면서 부족한 영역을 padding하여 224 x 224 입력을 만든다. 현재 학습 스크립트, centroid 생성 스크립트, 예측 스크립트가 모두 같은 방식으로 이미지를 처리한다.

## 4. 출력 명세

| 항목 | 설명 |
|---|---|
| softmax 확률 | 9개 클래스 각각에 대한 분류 확률 |
| classification top-1 | softmax 확률이 가장 높은 클래스 |
| embedding similarity | 입력 이미지 embedding과 각 클래스 centroid 간 cosine similarity |
| similarity top-1 | 유사도가 가장 높은 클래스 |
| similarity threshold | similarity top-1 클래스가 통과해야 하는 클래스별 임계값 |
| 최종 결과 | known이면 클래스명, 조건 미통과 시 `unknown` |

## 5. 클래스 목록

라벨 순서는 모델 출력 index, encoder 클래스, centroid 클래스 순서와 동일하다.

| Index | 클래스 |
|---:|---|
| 0 | 8자형 |
| 1 | 마름모형 |
| 2 | 물방울형 |
| 3 | 사각형 |
| 4 | 삼각형 |
| 5 | 원형 |
| 6 | 장방형 |
| 7 | 타원형 |
| 8 | 팔각형 |

## 6. 모델 구조

| Layer | Type | Output Shape | Params |
|---|---|---:|---:|
| `input_layer_2` | InputLayer | `(None, 224, 224, 3)` | 0 |
| `efficientnetb0` | Functional | `(None, 7, 7, 1280)` | 4,049,571 |
| `global_average_pooling2d` | GlobalAveragePooling2D | `(None, 1280)` | 0 |
| `dropout` | Dropout | `(None, 1280)` | 0 |
| `dense` | Dense, softmax | `(None, 9)` | 11,529 |

| 항목 | 값 |
|---|---:|
| Total params | 4,061,100 |
| Trainable params | 1,507,689 |
| Non-trainable params | 2,553,411 |
| 모델 입력 shape | `(None, 224, 224, 3)` |
| 모델 출력 shape | `(None, 9)` |
| 임베딩 벡터 차원 | 1280 |

## 7. 학습 설정

학습 스크립트 기준 주요 설정은 다음과 같다.

| 항목 | 값 |
|---|---|
| Backbone | EfficientNetB0 |
| 사전학습 가중치 | ImageNet |
| include_top | False |
| 입력 크기 | 224 x 224 x 3 |
| 전처리 | `resize_with_pad` 후 EfficientNet `preprocess_input` |
| Batch size | 32 |
| Seed | 42 |
| Label encoder | `sklearn.preprocessing.LabelEncoder` |
| Split | `StratifiedGroupKFold(n_splits=5, shuffle=True, random_state=42)` |
| Group 기준 | `item_seq` |
| Loss | `sparse_categorical_crossentropy` |
| Metric | `accuracy` |
| Stage 1 optimizer | Adam 기본 설정 |
| Stage 1 epochs | 15 |
| Stage 2 optimizer | Adam, learning rate `1e-5` |
| Stage 2 fine-tuning | EfficientNetB0 마지막 30개 layer만 trainable |
| Stage 2 epochs | 10 |
| Best model 기준 | `val_accuracy` 최대 모델 저장 |

학습 augmentation:

- `RandomFlip("horizontal")`
- `RandomRotation(0.1)`
- `RandomZoom(0.1)`
- `RandomTranslation(0.05, 0.05)`
- `RandomContrast(0.1)`

## 8. 저장된 학습 이력 요약

| 단계 | 마지막 accuracy | 마지막 loss | 마지막 val_accuracy | 마지막 val_loss | learning_rate |
|---|---:|---:|---:|---:|---:|
| Stage 1 learning | 0.961649 | 0.112621 | 0.962982 | 0.097187 | 0.000500 |
| Stage 2 tuning | 0.966621 | 0.098082 | 0.963501 | 0.093894 | 0.000010 |

저장된 history CSV 기준 최고 validation accuracy:

| 단계 | 최고 val_accuracy |
|---|---:|
| Stage 1 learning | 0.963915 |
| Stage 2 tuning | 0.963501 |

## 9. Centroid Similarity Index

`pill_shape_centroids.pkl`은 분류 모델의 `GlobalAveragePooling2D` 출력을 embedding으로 사용한다.

| 항목 | 값 |
|---|---|
| index version | 1 |
| reference image count | 48,261 |
| centroid shape | `(9, 1280)` |
| thresholds shape | `(9,)` |
| threshold 산출 방식 | 클래스별 leave-one-out centroid similarity의 5 percentile |
| similarity 계산 | L2 normalize 후 cosine similarity |
| centroid 생성 전처리 | `resize_with_pad` 후 EfficientNet `preprocess_input` |
| 모델 일치성 확인 | centroid index의 `model_mtime_ns`와 최종 모델 파일 mtime 비교 |

클래스별 centroid 통계:

| 클래스 | 기준 이미지 수 | similarity min | similarity mean | threshold |
|---|---:|---:|---:|---:|
| 8자형 | 316 | 0.11783412 | 0.73443562 | 0.56426609 |
| 마름모형 | 189 | 0.25292575 | 0.72585267 | 0.36724976 |
| 물방울형 | 183 | 0.24955091 | 0.63586837 | 0.37894052 |
| 사각형 | 510 | 0.09700916 | 0.62350160 | 0.30049878 |
| 삼각형 | 462 | 0.16326666 | 0.56971735 | 0.34399620 |
| 원형 | 19,032 | 0.00352833 | 0.54692674 | 0.36332276 |
| 장방형 | 13,271 | 0.14603272 | 0.58129323 | 0.32805812 |
| 타원형 | 13,806 | -0.02014393 | 0.57133526 | 0.26398769 |
| 팔각형 | 492 | 0.26798055 | 0.74265027 | 0.56510967 |

## 10. 추론 및 최종 판정 로직

추론은 `pill_shape_predict.py` 기준으로 다음 순서로 수행된다.

1. 입력 이미지를 RGB로 decode한다.
2. `tf.image.resize_with_pad(image, 224, 224)`로 원본 비율을 유지하며 224 x 224 입력을 만든다.
3. EfficientNet `preprocess_input`을 적용한다.
4. 최종 분류 모델에서 softmax 확률을 계산한다.
5. 같은 입력을 embedding model에 넣어 `GlobalAveragePooling2D` 출력 1280차원 embedding을 얻는다.
6. 입력 embedding과 9개 class centroid를 L2 normalize한다.
7. `centroids @ embedding`으로 클래스별 cosine similarity를 계산한다.
8. softmax top-1 클래스와 similarity top-1 클래스를 구한다.
9. 다음 세 조건을 모두 만족하면 known으로 판정한다.

Known 판정 조건:

| 조건 | 기준 |
|---|---|
| softmax 확률 조건 | classification top-1 확률 >= `0.60` |
| 유사도 조건 | similarity top-1 유사도 >= 해당 클래스 threshold |
| 클래스 일치 조건 | classification top-1 클래스 == similarity top-1 클래스 |

최종 결과:

- 세 조건을 모두 만족하면 classification top-1 클래스명을 반환한다.
- 하나라도 만족하지 못하면 `unknown`을 반환한다.

Unknown 사유:

| 사유 | 의미 |
|---|---|
| `softmax < 60.00%` | softmax top-1 확률이 최소 기준 미만 |
| `similarity below class threshold` | similarity top-1 유사도가 해당 클래스 임계값 미만 |
| `classification/similarity classes disagree` | softmax top-1과 similarity top-1 클래스가 다름 |

## 11. 파일 간 정합성

확인 결과:

- 모델 출력 클래스 수: 9
- encoder 클래스 수: 9
- centroid class_names 수: 9
- centroid shape 첫 번째 차원: 9
- threshold 수: 9
- encoder 클래스 순서와 centroid 클래스 순서: 일치
- centroid index의 `model_mtime_ns`: `1782232220793448700`
- 최종 모델 파일 mtime: `1782232220793448700`
- centroid index의 `model_mtime_ns`와 최종 모델 파일 mtime: 일치

따라서 현재 `efficientnetb0_pill_shape_best_tuning.keras`, `pill_shape_encoder.pkl`, `pill_shape_centroids.pkl` 세 파일은 서로 같은 클래스 체계와 모델 버전을 기준으로 생성된 것으로 판단된다.

## 12. 변경 반영 사항

이번 명세서에는 다음 변경 사항을 반영했다.

- 예측 전처리를 `tf.image.resize`가 아닌 `tf.image.resize_with_pad` 기준으로 수정
- centroid index 재생성 결과 반영
- 클래스별 similarity threshold와 통계값 갱신
- 기존 `pill_shape_model_proto.md`의 문서 구조를 참고하되, 깨진 한글 텍스트는 재작성

## 13. 운영 시 주의사항

- `pill_shape_encoder.pkl`과 `pill_shape_centroids.pkl`은 Python pickle 파일이므로 신뢰 가능한 파일만 로드해야 한다.
- 최종 모델 파일이 변경되면 `pill_shape_centroids.pkl`의 `model_mtime_ns` 검증에서 실패할 수 있으며, centroid index를 다시 생성해야 한다.
- `미등록`은 encoder의 학습 클래스가 아니며, 최종 판정에서 known 조건을 통과하지 못한 경우 `unknown`으로 처리되는 개념이다.
- TensorFlow 2.11 이상 native Windows 환경에서는 일반 CUDA GPU가 사용되지 않을 수 있으며, 현재 환경에서는 CPU 추론 경고가 출력되었다.
