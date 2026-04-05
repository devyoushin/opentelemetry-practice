# 백엔드 연동 가이드

OTel Collector에서 수집한 Traces / Metrics / Logs를 각 백엔드로 연동합니다.

---

## 전체 구성

```
[OTel Collector]
   ├── traces  ──→ Grafana Tempo    (분산 트레이싱)
   ├── metrics ──→ Prometheus       (메트릭, 알림)
   └── logs    ──→ Loki             (로그)
                       │
                  [Grafana]         (통합 시각화)
```

---

## 1. Grafana Tempo (트레이스 백엔드)

### Tempo 설치

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install tempo grafana/tempo \
  --namespace monitoring \
  --set tempo.storage.trace.backend=local
```

### Collector → Tempo 연동

```yaml
# Collector config
exporters:
  otlp:
    endpoint: tempo.monitoring:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, k8sattributes, batch]
      exporters: [otlp]
```

### Grafana에서 Tempo 데이터소스 추가

```
Grafana → Configuration → Data Sources → Add → Tempo
  URL: http://tempo.monitoring:3200
  Trace to logs:
    Data source: Loki
    Tags: [service.name, k8s.namespace.name]
    Map tag names: service.name → app
  Service graph:
    Data source: Prometheus
```

### Tempo에서 트레이스 조회 (TraceQL)

```
# Grafana → Explore → Tempo

# 에러 트레이스 조회
{ status = error }

# 특정 서비스의 느린 요청
{ resource.service.name = "my-app" && duration > 1s }

# HTTP 500 에러
{ span.http.status_code = 500 }

# 특정 사용자의 요청 추적
{ span.user.id = "1001" }
```

---

## 2. Prometheus (메트릭 백엔드)

### Collector → Prometheus 연동

Collector의 Prometheus exporter가 `/metrics`를 노출하고 Prometheus가 스크레이핑합니다.

```yaml
# Collector config
exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"

service:
  pipelines:
    metrics:
      receivers: [otlp, kubeletstats]
      processors: [memory_limiter, resource, batch]
      exporters: [prometheus]
```

Prometheus scrape 설정:
```yaml
# prometheus-values.yaml
extraScrapeConfigs: |
  - job_name: otelcol-metrics
    static_configs:
      - targets: ['otelcol-deployment.monitoring:8889']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
```

### Grafana에서 Prometheus 데이터소스 추가

```
Grafana → Configuration → Data Sources → Add → Prometheus
  URL: http://prometheus-server.monitoring:80
  Exemplars:
    Internal link: Tempo
    Label name: trace_id       # Prometheus 메트릭의 trace_id → Tempo 연결
```

---

## 3. Loki (로그 백엔드)

### Collector → Loki 연동

```yaml
# Collector config
exporters:
  loki:
    endpoint: http://loki.monitoring:3100/loki/api/v1/push

service:
  pipelines:
    logs:
      receivers: [filelog, otlp]
      processors: [memory_limiter, k8sattributes, resource, batch]
      exporters: [loki]
```

### Grafana에서 Loki 데이터소스 추가

```
Grafana → Configuration → Data Sources → Add → Loki
  URL: http://loki.monitoring:3100
  Derived fields:
    Name: TraceID
    Regex: "trace_id":"([a-f0-9]+)"
    Internal link: Tempo
    URL: ${__value.raw}
```

---

## 4. Grafana 통합 대시보드

세 가지 신호를 하나의 Grafana에서 봅니다.

### 추천 대시보드 임포트

```
Grafana → Dashboards → Import → ID 입력
```

| 대시보드 | ID | 내용 |
|----------|----|------|
| OpenTelemetry Collector | 15983 | Collector 운영 메트릭 |
| Kubernetes / Compute Resources | 15760 | 노드/Pod 리소스 |
| Tempo / Traces | 16611 | 트레이스 현황 |

### 변수 활용 대시보드 예시

```
서비스 선택: $service = label_values(traces_spanmetrics_calls_total, service)
환경 선택: $env = label_values(up, env)
```

---

## 5. 신호 간 연동 (Navigate between signals)

| 출발 | 도착 | 방법 |
|------|------|------|
| Tempo 트레이스 | Loki 로그 | Span의 "Logs" 버튼 클릭 (trace_id 자동 검색) |
| Loki 로그 | Tempo 트레이스 | Derived Fields의 trace_id 클릭 |
| Prometheus 메트릭 | Tempo 트레이스 | Exemplar 포인트 클릭 |
| Tempo 서비스 그래프 | Prometheus 메트릭 | 서비스 노드 클릭 |

---

## 6. 전체 연동 확인

```bash
# 각 컴포넌트 포트 포워딩
kubectl port-forward svc/grafana -n monitoring 3000:80
kubectl port-forward svc/tempo -n monitoring 3200:3200
kubectl port-forward svc/prometheus-server -n monitoring 9090:80
kubectl port-forward svc/loki -n monitoring 3100:3100

# Collector 수집 상태 확인
kubectl port-forward -n monitoring deployment/otelcol-deployment 8888:8888
curl http://localhost:8888/metrics | grep -E "receiver_accepted|exporter_sent"
```

---

## 참고

- [공식문서 - Grafana Tempo](https://grafana.com/docs/tempo/latest/)
- [공식문서 - Exemplars](https://grafana.com/docs/grafana/latest/fundamentals/exemplars/)
- [Grafana 대시보드 검색](https://grafana.com/grafana/dashboards/?search=opentelemetry)
