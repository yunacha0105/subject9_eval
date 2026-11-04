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
- Train 1,034장, Validation 133장, Test 128장
- SHA-256 해시 비교 결과 데이터 분할 간 동일 파일 중복 0건
- 데이터셋의 공개·배포 권한 확인 후 공유

## 3. 모델 준비
- `training/finetune.py` 실행
- Head-only 모델 저장: `artifacts/beans_resnet18/`
- Full Fine-tuning 모델 저장: `artifacts/beans_resnet18_full/`
- `.env`: `MODEL_PATH=artifacts/beans_resnet18`

## 4. 구동 순서
1. `docker compose up -d --build`
2. `docker compose exec redis redis-cli ping` 결과 `PONG` 확인
3. `docker compose ps`로 web, worker, redis, postgres 실행 상태 확인
4. 동기 화면: http://127.0.0.1:8000/vision/sync/
5. 비동기 화면: http://127.0.0.1:8000/vision/async/
6. API 문서: http://127.0.0.1:8000/docs

## 5. 확인할 응답
- `/health`: `{"status":"ok"}`
- 동기 POST: HTTP 200, `top1`, `all`, `cache_hit`, `inference_ms`
- 비동기 POST: HTTP 202, `job_id`; GET: `status=done`
- PostgreSQL: `image_jobs`에 비동기 작업 기록
- Redis: 동일 이미지 반복 요청 시 Cache HIT 확인

## 6. 결과물 위치
- 개별 보고서: `reports/01_*.md` ~ `reports/08_*.md`
- 성능 측정 원본: `reports/data/miss_paired_5runs.csv`
- Head-only Test 결과: `reports/data/test_metrics.json`
- Full Fine-tuning Test 결과: `reports/data/full_test_metrics.json`
- 실행 증빙: `reports/evidence/`
- Head-only 학습 결과: `artifacts/beans_resnet18/training_report.json`
- Full Fine-tuning 학습 결과: `artifacts/beans_resnet18_full/training_report.json`

## 7. 제출 및 보안
- `.env`, 비밀번호, API 키는 GitHub에 업로드하지 않음
- 데이터셋·가중치 대용량 파일은 허용된 제출 경로로 별도 전달
- GitHub에 제외한 파일의 확보·배치 절차를 명시