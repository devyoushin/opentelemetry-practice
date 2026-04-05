# 샘플링(Sampling) 가이드

트래픽이 많을 때 모든 트레이스를 저장하면 비용이 폭발합니다. 샘플링으로 중요한 트레이스만 선별합니다.

---

## 샘플링 종류

```
요청 시작 시점에 결정          데이터 수집 후 결정
       │                              │
[Head Sampling]              [Tail Sampling]
  빠르고 단순                  정확하지만 복잡
  에러/느린 요청 놓칠 수 있음   에러/느린 요청 정확히 포착
```

---

## 1. Head Sampling (앱/SDK 레벨)

요청이 시작될 때 샘플링 여부를 결정합니다. 이후 모든 하위 Span은 동일한 결정을 따릅니다.

### TraceIdRatio — 비율 기반

```yaml
# Instrumentation CR
spec:
  sampler:
    type: parentbased_traceidratio
    argument: "0.1"    # 10%만 샘플링
```

`parentbased_traceidratio`는 부모 Span의 결정을 따릅니다.
- 부모가 샘플링됨 → 자식도 샘플링
- 부모가 샘플링 안 됨 → 자식도 안 함

### AlwaysOn / AlwaysOff

```yaml
spec:
  sampler:
    type: always_on     # 모든 트레이스 저장 (개발/테스트용)
    # type: always_off  # 아무것도 저장 안 함
```

---

## 2. Tail Sampling (Collector 레벨, 권장)

Span이 모두 모인 뒤에 Trace 전체를 보고 샘플링 여부를 결정합니다.
에러, 지연 시간 등을 기준으로 **중요한 트레이스를 정확히 포착**할 수 있습니다.

```
[DaemonSet Collector]   →  [Gateway Collector (Tail Sampling)]  →  [Tempo]
   (전부 수집)               (정책에 따라 일부만 전달)
```

> **주의**: Tail Sampling은 같은 Trace의 모든 Span이 **동일한 Collector 인스턴스**에 도달해야 합니다.
> 여러 Collector가 있다면 `loadbalancingexporter`로 trace_id 기반 라우팅을 먼저 해야 합니다.

### Tail Sampling Processor 설정

```yaml
processors:
  tail_sampling:
    decision_wait: 10s          # Trace 완성까지 기다리는 시간
    num_traces: 50000           # 메모리에 보관할 최대 Trace 수
    expected_new_traces_per_sec: 10

    policies:
      # 정책 1: 에러가 있는 Trace는 반드시 저장
      - name: errors-policy
        type: status_code
        status_code:
          status_codes: [ERROR]

      # 정책 2: 응답 시간 1초 초과 Trace 저장
      - name: slow-traces-policy
        type: latency
        latency:
          threshold_ms: 1000

      # 정책 3: 나머지는 10%만 샘플링
      - name: probabilistic-policy
        type: probabilistic
        probabilistic:
          sampling_percentage: 10

      # 정책 4: 특정 서비스는 100% 저장
      - name: critical-service-policy
        type: string_attribute
        string_attribute:
          key: service.name
          values: [payment-service, auth-service]

      # 정책 5: 복합 조건 (AND)
      - name: composite-policy
        type: and
        and:
          and_sub_policy:
            - name: not-health-check
              type: string_attribute
              string_attribute:
                key: http.route
                values: [/health, /ready]
                invert_match: true
            - name: probabilistic
              type: probabilistic
              probabilistic:
                sampling_percentage: 20
```

---

## 3. Loadbalancing Exporter (Tail Sampling 필수)

Collector가 여러 개일 때 같은 trace_id가 동일한 Collector로 가도록 합니다.

```yaml
# DaemonSet Collector → Gateway Collector로 보낼 때
exporters:
  loadbalancing:
    routing_key: traceID       # trace_id 기준으로 라우팅
    protocol:
      otlp:
        tls:
          insecure: true
    resolver:
      k8s:
        service: otelcol-gateway-headless    # Gateway Collector Headless Service
        ports: [4317]

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [loadbalancing]   # Gateway로 전달 (샘플링은 Gateway에서)
```

---

## 4. 샘플링 정책 선택 가이드

```
트래픽 적음 (개발/테스트)
  → always_on (100% 저장)

트래픽 많음, 단순하게
  → parentbased_traceidratio (1~10%)

트래픽 많음, 에러/슬로우 쿼리를 반드시 잡아야 함
  → tail_sampling (에러 100% + 나머지 X%)
```

| 정책 | 비용 | 정확도 | 복잡도 |
|------|------|--------|--------|
| `always_on` | 높음 | 100% | 낮음 |
| `traceidratio` | 낮음 | 무작위 | 낮음 |
| `tail_sampling` | 중간 | 높음 (에러 보장) | 높음 |

---

## 5. 샘플링 상태 확인

```bash
# Tail Sampling 결정 현황 메트릭
curl http://localhost:8888/metrics | grep otelcol_processor_tail_sampling

# otelcol_processor_tail_sampling_count_traces_sampled     — 저장된 trace 수
# otelcol_processor_tail_sampling_count_traces_not_sampled — 버려진 trace 수
# otelcol_processor_tail_sampling_sampling_decision_timer_latency — 결정 지연
```

---

## 참고

- [공식문서 - Sampling](https://opentelemetry.io/docs/concepts/sampling/)
- [공식문서 - Tail Sampling Processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor)
- [공식문서 - Loadbalancing Exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/loadbalancingexporter)
