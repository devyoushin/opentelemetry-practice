# OpenTelemetry 컨벤션 규칙

## Span 네이밍
- HTTP 서버: `{HTTP method} {route}` (예: `GET /api/users/{id}`)
- HTTP 클라이언트: `{HTTP method}` (예: `GET`)
- DB 쿼리: `{db.system} {operation}` (예: `postgresql query`)
- 메시지 소비: `{destination} receive`
- 내부 작업: `{module}.{operation}` (예: `auth.validate_token`)

## Span Attribute 표준 (Semantic Conventions)
```yaml
# HTTP
http.method, http.url, http.status_code, http.target
# DB
db.system, db.name, db.statement, db.operation
# 메시징
messaging.system, messaging.destination, messaging.operation
# 공통
service.name, service.version, deployment.environment
```

## Collector 파이프라인 컨벤션
```yaml
# 필수 processor 순서
processors:
  - memory_limiter    # 항상 첫 번째
  - resourcedetection # 환경 정보 자동 감지
  - batch             # 마지막 (배치 효율화)
```

## 리소스 속성 필수 설정
```yaml
resource:
  attributes:
    service.name: "my-service"           # 필수
    service.version: "1.0.0"             # 필수
    deployment.environment: "production" # 필수
    service.namespace: "payment"         # 권장
```

## Sampling 정책
- 개발: 100% (all)
- 스테이징: 10% (head sampling)
- 프로덕션: 1-5% (tail sampling, 오류/슬로우 쿼리 100%)

## Metric 네이밍
- OTLP 메트릭은 Prometheus 네이밍 변환 규칙 준수
- Unit 포함: `http.server.duration` (ms), `http.server.request.size` (bytes)
- 히스토그램 사용: 지연 시간, 크기 측정에 우선 적용
