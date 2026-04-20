새 OpenTelemetry 운영 런북을 생성합니다.

**사용법**: `/new-runbook <작업명>`  **예시**: `/new-runbook Collector 파이프라인 변경`

작업 유형: Collector 설정 변경, 샘플링 정책 조정, 백엔드 연동 변경, 계측 롤아웃

런북 포함 내용:
- 사전 체크리스트 (현재 수집 메트릭/트레이스 수)
- 단계별 kubectl/helm 명령어
- otel-cli 또는 curl로 파이프라인 검증
- 롤백 절차
