새 OpenTelemetry 가이드 문서를 생성합니다.

**사용법**: `/new-doc <주제명>`  **예시**: `/new-doc tail-sampling`

주제 분류: collector, instrumentation, tracing, metrics, logging, sampling, backend-integration

`<주제명>-guide.md` 생성 시 포함 내용:
- CLAUDE.md 환경 설정 반영 (EKS, OTel Collector 버전)
- Collector 파이프라인 YAML 예시
- 계측 코드 예시 (Java/Go/Python)
- 백엔드 연동 설정 (Tempo/Prometheus/Loki)
- 트러블슈팅 섹션
