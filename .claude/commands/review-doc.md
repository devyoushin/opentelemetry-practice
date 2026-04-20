OpenTelemetry 가이드 문서를 검토합니다.

**사용법**: `/review-doc <파일 경로>`  **예시**: `/review-doc collector-guide.md`

검토 기준:
- Collector 파이프라인: receivers→processors→exporters 순서
- 메모리 제한: memory_limiter processor 포함 여부
- 배치 설정: batch processor 포함 여부 (성능)
- 샘플링: 트레이스 샘플링 전략 (head/tail)
- 보안: TLS 설정, 인증 (API key, OIDC)
- 문서: 파이프라인 다이어그램, 설정 예시 포함
