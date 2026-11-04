# API 명세서

| API ID | Method | URL                         | 요청 형태                           | 성공 HTTP | 주요 응답                                                  | 실패 HTTP            |
| ------ | ------ | --------------------------- | ------------------------------- | ------- | ------------------------------------------------------ | ------------------ |
| API-01 | GET    | `/health`                   | 없음                              | 200     | `status`                                               | 서버 미실행은 HTTP 응답 없음 |
| API-02 | POST   | `/api/sync/classify/`       | `multipart/form-data` / `image` | 200     | `model_id`, `top1`, `all`, `inference_ms`, `cache_hit` | 400, 413           |
| API-03 | POST   | `/api/async/classify/`      | `multipart/form-data` / `image` | 202     | `job_id`, `status`, `check_url`                        | 400, 413, 503      |
| API-04 | GET    | `/api/async/jobs/{job_id}/` | 정수형 `job_id`                    | 200     | `status`, `top1`, `all`, `error_message` 등             | 404                |
| WEB-01 | GET    | `/vision/sync/`             | 없음                              | 200     | HTML                                                   | —                  |
| WEB-02 | GET    | `/vision/async/`           | 없음                              | 200     | HTML                                                   | —                  |

**API-02: 동기 분류 상세**

| 항목 | 내용 |
|---|---|
| URL | `/api/sync/classify/` |
| Method | POST |
| Content-Type | `multipart/form-data` |
| 요청 필드 | `image`: JPEG/PNG/WEBP, 5MB 이하 |
| 성공 코드 | 200 OK |
| 오류 코드 | 400 잘못된 이미지, 413 초과 크기 |

**API-02 성공 응답 JSON**

```json
{
  "model_id": "artifacts/beans_resnet18",
  "top1": {
    "label": "healthy",
    "score": 0.6200540661811829
  },
  "all": [
    {
      "label": "healthy",
      "score": 0.6200540661811829
    },
    {
      "label": "bean_rust",
      "score": 0.2988567054271698
    },
    {
      "label": "angular_leaf_spot",
      "score": 0.08108920603990555
    }
  ],
  "inference_ms": 19.69,
  "cache_hit": false
}
```

**API-03 접수 응답 JSON**

```json
{
  "job_id": 13,
  "status": "queued",
  "check_url": "/api/async/jobs/13/"
}
```

**API-04 완료 응답 JSON**

```json
{
  "job_id": 13,
  "status": "done",
  "model_id": "artifacts/beans_resnet18",
  "top1": {
    "label": "healthy",
    "score": 0.6200540661811829
  },
  "all": [
    {
      "label": "healthy",
      "score": 0.6200540661811829
    },
    {
      "label": "bean_rust",
      "score": 0.2988567054271698
    },
    {
      "label": "angular_leaf_spot",
      "score": 0.08108920603990555
    }
  ],
  "inference_ms": null,
  "cache_hit": true,
  "error_message": null
}
```

**API-02 오류 응답 JSON**

```json
{
  "detail": "올바른 이미지 파일이 아닙니다."
}
```

**API-04 오류 응답 JSON**

```json
{
  "detail": "해당 작업이 없습니다."
}
```

**실행 근거표**

| 검증 ID | 테스트 입력 | 실제 상태코드 | 첨부 증빙 |
|---|---|---:|---|
| API-01-T01 | GET `/health` | 200 | `evidence/health.json` |
| API-02-T01 | `healthy_test.6.jpg` | 200 | `evidence/sync_ui.png` |
| API-03-T01 | `healthy_test.6.jpg` | 202 | `evidence/worker_web_logs.txt` |
| API-04-T01 | 접수된 `job_id=13` | 200 | `evidence/async_ui.png` |
| API-02-T02 | 잘못된 이미지 파일 | 400 | `evidence/real_error_responses.json` |
| API-02-T03 | 5MB 초과 파일 | 413 | `evidence/real_error_responses.json` |
| API-04-T02 | 없는 `job_id=99999999` | 404 | `evidence/real_error_responses.json` |

**검증 결과**

- 동기 API는 HTTP 200을 반환하였으며, `healthy_test.6.jpg`를 `healthy`로 분류하였다.
- 동기 API 응답시간은 30ms, 모델 추론시간은 19.69ms로 측정되었다.
- 비동기 API는 HTTP 202로 작업을 접수하였으며, `job_id=13`의 최종 상태는 `done`으로 확인되었다.
- 비동기 작업의 접수시간은 15ms, 최종 결과 확인시간은 1,032ms로 측정되었다.
- 비동기 작업은 Redis Cache HIT로 처리되어 모델 추론을 생략하였다.
- 잘못된 이미지, 파일 크기 초과, 존재하지 않는 작업번호에 대해 각각 HTTP 400, 413, 404 응답을 확인하였다.
- API 응답의 `all` 필드와 PostgreSQL의 `all_scores` 컬럼을 구분하였다.