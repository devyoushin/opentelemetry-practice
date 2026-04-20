OpenTelemetry 트러블슈팅 케이스를 추가합니다.

**사용법**: `/add-troubleshooting <증상 설명>`  **예시**: `/add-troubleshooting 트레이스 데이터 누락`

형식:
```markdown
### <증상>
**원인**: <근본 원인>
**확인**:
\`\`\`bash
kubectl logs -n monitoring -l app.kubernetes.io/name=opentelemetry-collector
curl http://collector:8888/metrics | grep otelcol_
\`\`\`
**해결**: <해결 방법>
```
