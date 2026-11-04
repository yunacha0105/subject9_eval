# FastAPI Beans ResNet18 동기·비동기 평가 프로젝트

## 1. 환경
- Windows 11 + WSL2 Ubuntu 26.04.1 LTS
- Python 3.14.4 (WSL 가상환경), Python 3.11 (Docker 컨테이너)
- Docker Desktop / Docker Compose
- Docker: Redis 6379, PostgreSQL 5432
- Docker Compose 서비스: FastAPI, Celery Worker, Redis, PostgreSQL

## 2. 데이터 준비
- 제공 원본 `validation/`의 각 클래스 이미지를 코드 실행용 `dataset/val/`에 배치
- `dataset/train/`, `dataset/val/`, `dataset/test/` 모두 동일 3클래스
  - `angular_leaf_spot`
  - `bean_rust`
  - `healthy`
- Train 1,034장, Validation 133장, Test 128장
- SHA-256 해시 비교 결과 데이터 분할 간 동일 파일 중복 0건
- 원본 데이터와 실행용 데이터의 일대일 동일성 및 유사 이미지 중복 여부는 별도 검증하지 않음
- 데이터셋의 공개·배포 권한 확인 후 공유

**데이터셋 폴더 구조**

```text
dataset/
├── train/
│   ├── angular_leaf_spot/
│   ├── bean_rust/
│   └── healthy/
├── val/
│   ├── angular_leaf_spot/
│   ├── bean_rust/
│   └── healthy/
└── test/
    ├── angular_leaf_spot/
    ├── bean_rust/
    └── healthy/
```

- 데이터셋은 GitHub 저장소에 포함하지 않으므로 별도로 확보하여 위 구조로 배치한다.

## 3. 모델 준비
- 제공 `training/finetune.py` 실행
- Head-only 모델 저장: `artifacts/beans_resnet18/`
- Full Fine-tuning 모델 저장: `artifacts/beans_resnet18_full/`
- 현재 API 서비스 모델: Head-only (`artifacts/beans_resnet18`)
- 데이터셋과 모델 가중치는 GitHub에 포함하지 않으므로 별도 확보가 필요하다.

**Head-only Fine-tuning 실행**

```bash
python -m training.finetune --data dataset --output artifacts/beans_resnet18 --epochs 3 --batch-size 8 --freeze-backbone
```

**Full Fine-tuning 실행**

```bash
python -m training.finetune --data dataset --output artifacts/beans_resnet18_full --epochs 3 --batch-size 8
```

- 두 실험 모두 제공 코드를 변경하지 않고 실행 옵션만 다르게 설정하였다.
- 현재 서비스는 Head-only 모델을 사용하며 Full Fine-tuning 결과는 별도 비교 실험으로 기록하였다.

**모델 폴더 구조**

```text
artifacts/
├── beans_resnet18/
│   ├── config.json
│   ├── model.safetensors
│   ├── preprocessor_config.json
│   └── training_report.json
└── beans_resnet18_full/
    ├── config.json
    ├── model.safetensors
    ├── preprocessor_config.json
    └── training_report.json
```

**환경변수 설정**

프로젝트 루트에서 아래 명령어를 실행한다.

```bash
cp .env.example .env
```

`.env`의 모델 경로를 확인한다.

```env
MODEL_PATH=artifacts/beans_resnet18
```

- `.env`는 GitHub에 업로드하지 않는다.
- 모델 경로를 변경하면 FastAPI와 Celery Worker 컨테이너를 재생성한다.

```bash
docker compose up -d --force-recreate web worker
```

## 4. 구동 순서
1. 데이터셋 및 학습된 모델 파일을 지정된 폴더에 배치한다.
2. `.env.example`을 복사하여 `.env`를 생성하고 `MODEL_PATH`를 확인한다.
3. Docker Compose 서비스를 실행한다.

```bash
docker compose up -d --build
```

4. Redis 연결을 확인한다.

```bash
docker compose exec redis redis-cli ping
```

정상 응답:

```text
PONG
```

5. 컨테이너 실행 상태를 확인한다.

```bash
docker compose ps
```

6. 브라우저에서 아래 주소에 접속한다.
   - 동기 화면: http://127.0.0.1:8000/vision/sync/
   - 비동기 화면: http://127.0.0.1:8000/vision/async/
   - API 문서: http://127.0.0.1:8000/docs

## 5. 확인할 응답
- `/health`: `{"status":"ok"}`
- 동기 POST: HTTP 200, `top1`, `all`, `cache_hit`, `inference_ms`
- 비동기 POST: HTTP 202, `job_id`
- 비동기 GET: `status=done`, 분류 결과 확인
- PostgreSQL: `image_jobs`에 비동기 작업 상태 및 결과 기록
- Redis: 동일 이미지 반복 요청 시 Cache HIT 확인

**실제 검증 결과**
- 동기 API에서 `healthy_test.6.jpg`를 `healthy`로 분류하였다.
- 비동기 작업 `job_id=13`의 최종 상태가 `done`으로 확인되었다.
- 잘못된 이미지 업로드 HTTP 400, 파일 크기 초과 HTTP 413, 없는 작업번호 조회 HTTP 404를 확인하였다.

## 6. 결과물 위치
- 개별 보고서: `reports/01_*.md` ~ `reports/08_*.md`
- 성능 측정 원본: `reports/data/miss_paired_5runs.csv`
- Head-only Test 결과: `reports/data/test_metrics.json`
- Head-only 이미지별 예측: `reports/data/test_predictions.csv`
- Full Fine-tuning Test 결과: `reports/data/full_test_metrics.json`
- Full Fine-tuning 이미지별 예측: `reports/data/full_test_predictions.csv`
- 실행 증빙: `reports/evidence/`
- Head-only 학습 결과 기록: `reports/evidence/head_training_report.json`
- Full Fine-tuning 학습 결과 기록: `reports/evidence/full_training_report.json`
- 실제 모델 학습 결과: `artifacts/beans_resnet18/training_report.json`, `artifacts/beans_resnet18_full/training_report.json` (별도 확보)

**주요 평가 결과**

| 항목 | Head-only | Full Fine-tuning |
|---|---:|---:|
| Best Validation Accuracy | 76.69% | 97.74% |
| Test Accuracy | 71.88% | 90.63% |
| Test 이미지 수 | 128 | 128 |
| Test 정답 수 | 92 | 116 |

**동기·비동기 성능 비교 (Cache MISS, 5회 중앙값)**

| 측정 항목 | 중앙값 |
|---|---:|
| Sync HTTP 완료 | 22.499ms |
| Async HTTP 접수 | 7.429ms |
| Async 최종 확인 | 62.997ms |

- 비동기는 HTTP 접수시간이 짧지만 최종 결과 확인시간은 더 길게 측정되었다.
- Cache HIT 대 HIT의 동일 조건 반복 성능 비교는 수행하지 않았다.

## 7. 제출 및 보안
- `.env`, 비밀번호, API 키는 GitHub에 업로드하지 않는다.
- `.gitignore`에 따라 `dataset/`, `artifacts/*`, `.env`는 GitHub 추적에서 제외한다.
- 데이터셋·가중치 대용량 파일은 사용·배포 권한을 확인한 뒤 허용된 경로로 별도 전달한다.
- 데이터셋은 `dataset/train/val/test/`, 모델 가중치는 `artifacts/beans_resnet18/`에 배치해야 한다.
- GitHub에 제외된 데이터셋과 모델 가중치는 별도 제공받거나 학습 과정을 재실행하여 확보해야 한다.
- 제출된 코드와 보고서만으로는 모델 가중치 및 데이터셋을 복원할 수 없다.