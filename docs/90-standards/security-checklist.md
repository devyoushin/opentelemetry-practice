# 보안 체크리스트 — opentelemetry-practice

## OpenTelemetry Collector 보안

### 인증/인가
- [ ] Collector exporter에 TLS 설정 (백엔드 전송 암호화)
- [ ] OTLP 수신 포트(4317/4318) 내부망 제한
- [ ] Collector API 키/토큰 환경변수로 관리 (하드코딩 금지)
- [ ] K8s Secret으로 자격증명 관리

### 데이터 보안
- [ ] 민감 Span attribute 필터링 (password, token, card_number)
  ```yaml
  processors:
    attributes:
      actions:
        - key: password
          action: delete
  ```
- [ ] PII(개인식별정보) Span에 포함 금지
- [ ] Baggage에 민감 데이터 포함 금지

### 네트워크 보안
- [ ] OTLP/gRPC(4317), OTLP/HTTP(4318) 외부 노출 금지
- [ ] Collector → 백엔드 TLS 인증서 검증
- [ ] NetworkPolicy로 Collector 접근 제한

### SDK 보안
- [ ] SDK 초기화 시 exporter 엔드포인트 환경변수로 관리
- [ ] Sampling으로 민감 데이터 포함 트레이스 수집 최소화

## 정기 보안 점검 (월별)
- [ ] Span attribute에 민감 데이터 포함 여부 감사
- [ ] Collector 자격증명 순환
- [ ] OTel SDK/Collector 보안 업데이트 확인
- [ ] 불필요한 telemetry 수집 중단
