# OTel 메트릭 가이드

OTel SDK로 수집한 메트릭을 Collector를 통해 Prometheus로 전달하는 흐름을 다룹니다.

---

## 메트릭 데이터 모델

### 기본 계측기 (Instrument) 종류

| 계측기 | 타입 | 사용 예시 |
|--------|------|-----------|
| **Counter** | 단조 증가 (누적) | HTTP 요청 수, 에러 수 |
| **UpDownCounter** | 증가/감소 가능 | 현재 활성 연결 수, 큐 크기 |
| **Gauge** | 현재 순간 값 | CPU 사용률, 메모리 사용량 |
| **Histogram** | 값 분포 (bucket) | 응답 시간, 요청 크기 |

---

## SDK로 메트릭 직접 계측

### Python 예시

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader

# MeterProvider 설정
exporter = OTLPMetricExporter(endpoint="http://otelcol:4317", insecure=True)
reader = PeriodicExportingMetricReader(exporter, export_interval_millis=10000)
provider = MeterProvider(metric_readers=[reader])
metrics.set_meter_provider(provider)

meter = metrics.get_meter("my-app")

# Counter — HTTP 요청 수
request_counter = meter.create_counter(
    name="http.requests.total",
    description="Total HTTP requests",
    unit="1",
)

# Histogram — 응답 시간
request_duration = meter.create_histogram(
    name="http.request.duration",
    description="HTTP request duration",
    unit="ms",
)

# 사용 예시
def handle_request(method, path, status_code, duration_ms):
    request_counter.add(1, {
        "http.method": method,
        "http.route": path,
        "http.status_code": str(status_code),
    })
    request_duration.record(duration_ms, {
        "http.method": method,
        "http.route": path,
    })
```

### Go 예시

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/metric"
)

meter := otel.Meter("my-app")

// Counter
requestCount, _ := meter.Int64Counter(
    "http.requests.total",
    metric.WithDescription("Total HTTP requests"),
)

// Histogram
requestDuration, _ := meter.Float64Histogram(
    "http.request.duration",
    metric.WithUnit("ms"),
)

// 사용
requestCount.Add(ctx, 1, metric.WithAttributes(
    attribute.String("http.method", "GET"),
    attribute.Int("http.status_code", 200),
))
```

---

## Collector에서 Prometheus로 내보내기

### 방식 1: Prometheus Exporter (Collector가 메트릭 노출)

Collector가 `/metrics` 엔드포인트를 열고 Prometheus가 스크레이핑합니다.

```yaml
# Collector config
exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: otel              # 메트릭 이름 앞에 "otel_" 접두어 추가
    const_labels:
      cluster: my-eks-cluster

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

Prometheus scrape 설정:
```yaml
# Prometheus values.yaml
extraScrapeConfigs: |
  - job_name: otelcol
    static_configs:
      - targets: ['otelcol-deployment.monitoring:8889']
```

### 방식 2: OTLP → Prometheus Remote Write

Collector가 Prometheus에 직접 push합니다.

```yaml
exporters:
  prometheusremotewrite:
    endpoint: http://prometheus.monitoring:9090/api/v1/write
    tls:
      insecure: true
```

---

## Prometheus에서 OTel 메트릭 조회

OTel SDK가 보낸 메트릭은 Prometheus에서 다음과 같이 조회합니다.

```promql
# 초당 HTTP 요청 수 (Counter → rate 변환 필수)
rate(http_requests_total[5m])

# 서비스별 에러율
sum(rate(http_requests_total{http_status_code=~"5.."}[5m])) by (service_name)
/
sum(rate(http_requests_total[5m])) by (service_name)

# 응답 시간 P99 (Histogram)
histogram_quantile(0.99,
  sum(rate(http_request_duration_bucket[5m])) by (le, service_name)
)
```

---

## 자동 수집되는 Kubernetes 메트릭 (kubeletstats receiver)

별도 계측 없이 kubelet API에서 메트릭을 수집합니다.

```yaml
receivers:
  kubeletstats:
    collection_interval: 20s
    auth_type: serviceAccount
    endpoint: "${env:K8S_NODE_NAME}:10250"
    insecure_skip_verify: true
    metric_groups: [node, pod, container]
```

수집되는 주요 메트릭:

| 메트릭 | 설명 |
|--------|------|
| `k8s.pod.cpu.utilization` | Pod CPU 사용률 |
| `k8s.pod.memory.usage` | Pod 메모리 사용량 |
| `k8s.container.restarts` | 컨테이너 재시작 횟수 |
| `k8s.node.cpu.utilization` | 노드 CPU 사용률 |

---

## 참고

- [공식문서 - Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [공식문서 - Prometheus Exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/prometheusexporter)
- [공식문서 - kubeletstats receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/kubeletstatsreceiver)
