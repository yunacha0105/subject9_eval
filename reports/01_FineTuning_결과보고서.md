# ResNet18 콩잎 3클래스 Fine-tuning 결과보고서

- 평가 대상: 콩잎 질병 이미지 3종 분류
- 기준 모델: microsoft/resnet-18
- 학습 방식: Head-only (Backbone freeze)
- 입력 데이터: dataset/train, dataset/val
- 최종 보관 데이터: dataset/test (학습에 사용하지 않고 최종 성능 평가에 사용)
- 모델 저장 위치: artifacts/beans_resnet18/
- 실험 상태: 실행 완료

## 1. 데이터 구성
| 클래스               | Train | Validation | Test |
| ----------------- | ----: | ---------: | ---: |
| angular_leaf_spot |   345 |         44 |   43 |
| bean_rust         |   348 |         45 |   43 |
| healthy           |   341 |         44 |   42 |
| 합계                |  1034 |        133 |  128 |

- 원본 폴더: train / validation / test
- 코드 적용 폴더: train / val / test
- 데이터 누출 점검: SHA-256 해시 비교 결과 Train/Validation/Test 간 동일 파일 중복 0건

## 2. 실행 설정
| 파라미터 | 값 |
|---|---|
| base_model | microsoft/resnet-18 |
| epochs | 3 |
| batch_size | 8 |
| learning_rate | 0.0002 |
| freeze_backbone | True |
| output | artifacts/beans_resnet18 |
| seed | 42 (제공 코드) |

## 3. 학습 결과
| Epoch | Train Loss | Validation Accuracy |
| ----- | ---------: | ------------------: |
| 1     |     0.9924 |              0.6617 |
| 2     |     0.8108 |              0.7669 |
| 3     |     0.7078 |              0.7669 |

- Best Validation Accuracy: 0.7669
- Best 모델 저장: artifacts/beans_resnet18/
- Test Accuracy: 0.71875 (128장 중 92장 정답)
- 별도 Test 성능 분석: 수행 완료

### 3-1. Test 데이터 성능 평가

학습에 사용하지 않은 Test 데이터 128장을 대상으로 Head-only Fine-tuning 모델의 분류 성능을 평가하였다.

|평가 지표|결과|
|---|---|
|Test 이미지 수|128|
|정답 이미지 수|92|
|오분류 이미지 수|36|
|Accuracy|0.71875 (71.88%)|
|Macro Precision|0.7149|
|Macro Recall|0.7204|
|Macro F1-score|0.7109|

### 3-2. 혼동행렬 (Confusion Matrix)

|실제 클래스 \ 예측 클래스|angular_leaf_spot|bean_rust|healthy|
|---|---|---|---|
|angular_leaf_spot|**30**|7|6|
|bean_rust|13|**23**|7|
|healthy|0|3|**39**|

### 3-3. 클래스별 성능

|클래스|Precision|Recall|F1-score|
|---|---|---|---|
|angular_leaf_spot|0.6977|0.6977|0.6977|
|bean_rust|0.6970|0.5349|0.6053|
|healthy|0.7500|0.9286|0.8298|

### 3-4. 결과 분석

Test 데이터 128장 중 92장을 올바르게 분류하여 Accuracy 71.88%를 기록하였다.

healthy 클래스는 Recall 92.86%로 가장 높았으며, bean_rust 클래스는 Recall 53.49%로 가장 낮았다. 특히 실제 bean_rust 이미지 43장 중 13장을 angular_leaf_spot으로 잘못 분류하였다.

이를 통해 Head-only Fine-tuning 모델에서 bean_rust 클래스의 분류 성능이 상대적으로 낮음을 확인하였다.

- 평가 결과: data/test_metrics.json
- 이미지별 예측 결과: data/test_predictions.csv

## 4. 생성 파일 확인
- [x] config.json
- [x] model.safetensors
- [x] preprocessor_config.json
- [x] training_report.json

## 5. 서비스 연결 확인
- 동기 API 응답 라벨: healthy
- 응답 점수: 0.6201
- 정답 일치 여부: 일치 (healthy_test.6.jpg)
## 6. 결과 판정
- 완료: Head-only Fine-tuning 학습 및 모델 파일 생성
- 한계: Test Accuracy 0.71875, 총 128장 중 36장 오분류. 특히 bean_rust 클래스의 Recall이 0.5349로 낮게 나타남
- 증빙: evidence/head_training_report.json, evidence/dataset_check.txt


### 실행 명령 기록 — 기존 제공 코드 실행 옵션만 사용
```bash
python -m training.finetune --data dataset --output artifacts/beans_resnet18 --epochs 3 --batch-size 8 --freeze-backbone
```

### 학습 결과 JSON에서 옮겨 적을 항목

```json
{
  "base_model": "microsoft/resnet-18",
  "classes": ["angular_leaf_spot", "bean_rust", "healthy"],
  "train_count": 1034,
  "val_count": 133,
  "best_val_accuracy": 0.7669172932330827,
  "history": [
    {"epoch": 1, "loss": 0.9924, "val_accuracy": 0.6617},
    {"epoch": 2, "loss": 0.8108, "val_accuracy": 0.7669},
    {"epoch": 3, "loss": 0.7078, "val_accuracy": 0.7669}
  ]
}
```