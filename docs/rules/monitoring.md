# 모니터링 규칙 — opentelemetry-practice

## OpenTelemetry Collector 자체 모니터링

### 핵심 메트릭

```promql
# Collector 수신 Span 수
rate(otelcol_receiver_accepted_spans_total[5m])

# Collector 수신 거부 수 (오류)
rate(otelcol_receiver_refused_spans_total[5m])

# Exporter 실패 수
rate(otelcol_exporter_send_failed_spans_total[5m])

# 큐 사용률
otelcol_exporter_queue_size / otelcol_exporter_queue_capacity

# 메모리 사용량
otelcol_process_memory_rss

# 배치 크기 (p50)
histogram_quantile(0.50, rate(otelcol_processor_batch_batch_size_trigger_send_bucket[5m]))
```

### 알림 규칙 (권장)

| 알림 | 조건 | 심각도 |
|------|------|--------|
| CollectorDown | `up{job="otelcol"} == 0` | critical |
| CollectorExporterFailure | 내보내기 실패 > 0 | warning |
| CollectorQueueFull | 큐 사용률 > 80% | warning |
| CollectorMemoryHigh | 메모리 > 제한의 80% | warning |

## SLO 정의 (Collector 서비스)

| SLI | 목표 | 측정 방법 |
|-----|------|---------|
| Span 수집 성공률 | 99.5% | refused/(accepted+refused) |
| Span 전송 성공률 | 99% | export 실패율 |
| 처리 지연 | < 1초 | 배치 처리 시간 |

## 대시보드 구성 (Grafana)
- **Overview**: 수신/거부/전송 현황
- **Receivers**: 신호별 수신 속도
- **Processors**: 배치 크기, 처리 시간
- **Exporters**: 전송 성공/실패, 큐 상태
- **Resources**: CPU, 메모리, 고루틴 수
