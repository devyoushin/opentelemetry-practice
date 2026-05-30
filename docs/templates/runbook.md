# [알림명] — OpenTelemetry 런북

> 심각도: Critical / Warning / Info
> 담당팀: | 최종 수정: YYYY-MM-DD

## 알림 요약
<!-- 이 알림이 무엇을 감지하는지 한 문장으로 -->

## 알림 조건
<!-- 알림 트리거 조건 설명 -->

## 영향도
<!-- 이 알림이 발생했을 때 사용자/서비스에 미치는 영향 -->

## 즉시 확인 사항

```bash
# 1. Collector 상태 확인
kubectl get pods -n monitoring -l app=opentelemetry-collector

# 2. Collector 로그 확인
kubectl logs -n monitoring -l app=opentelemetry-collector --tail=50

# 3. Collector 메트릭 확인 (내부 메트릭)
curl http://collector:8888/metrics | grep otelcol
```

## 진단 단계

### 1단계: Collector 파이프라인 확인
```bash
# Collector debug exporter 활용
# otelcol_receiver_accepted_spans 메트릭으로 수신 여부 확인
```

### 2단계: 원인 분석
<!-- 일반적인 원인 목록 -->

### 3단계: 해결 조치
<!-- 단계별 해결 방법 -->

## 에스컬레이션
- 15분 내 해결 불가 시: 팀 리드 호출
- 전체 텔레메트리 중단 시: 인시던트 선언

## 참고
- 관련 Grafana 대시보드:
- 관련 문서:
