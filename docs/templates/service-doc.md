# [서비스명] — OpenTelemetry 계측 문서

> 작성일: YYYY-MM-DD | 작성자: | 검토자:

## 개요
<!-- 이 서비스의 OTel 계측 목적과 수집 신호 -->

## 계측 구성

| 항목 | 값 |
|------|-----|
| OTel SDK 버전 | |
| 계측 언어 | |
| Collector 배포 | DaemonSet / Deployment / Sidecar |
| 백엔드 (Traces) | Jaeger / Tempo |
| 백엔드 (Metrics) | Prometheus / Mimir |
| 백엔드 (Logs) | Loki |

## 계측 신호

| 신호 | 수집 방식 | 백엔드 | 상태 |
|------|---------|--------|------|
| Traces | Auto / Manual | | |
| Metrics | Auto / Manual | | |
| Logs | | | |

## Collector 파이프라인

```yaml
# 핵심 Collector 설정 요약
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
```

## 주요 Span 설계

| Span 이름 | 타입 | 주요 Attribute | 설명 |
|-----------|------|--------------|------|
| | | | |

## 운영 체크리스트
- [ ] Collector 정상 실행 확인
- [ ] Trace 백엔드 도착 확인
- [ ] 메트릭 수집 확인
- [ ] Sampling 비율 적절성 확인

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|------|------|----------|
| | | |

## 관련 문서
- 런북:
- Collector 설정:
- Grafana 대시보드:
