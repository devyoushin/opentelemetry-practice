# OTel Collector 파이프라인 가이드

Collector는 OTel 스택의 핵심입니다. 앱과 백엔드 사이에 위치하여 telemetry 데이터를 수신, 처리, 전송합니다.

---

## 파이프라인 구조

```
[Receivers]          [Processors]          [Exporters]
    │                     │                     │
otlp ──────────────→ batch ──────────────→ otlp (Tempo)
prometheus_scrape ──→ memory_limiter ───→ prometheus
filelog ────────────→ resource ─────────→ loki
k8s_events ─────────→ attributes ──────→ debug (개발용)
```

모든 파이프라인은 **receivers → processors → exporters** 순서로 흐릅니다.

---

## 1. Receivers

데이터를 받아들이는 입구입니다.

### OTLP Receiver (가장 중요)

앱 SDK 또는 다른 Collector로부터 traces/metrics/logs를 받습니다.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317   # gRPC (기본)
      http:
        endpoint: 0.0.0.0:4318   # HTTP/protobuf
```

### Prometheus Receiver

기존 Prometheus 스크레이핑 방식으로 메트릭을 수집합니다.

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: my-app
          kubernetes_sd_configs:
            - role: pod
          relabel_configs:
            - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
              action: keep
              regex: "true"
```

### Filelog Receiver

파일(컨테이너 로그)에서 로그를 수집합니다.

```yaml
receivers:
  filelog:
    include:
      - /var/log/pods/*/*/*.log
    include_file_path: true
    operators:
      - type: container          # Kubernetes 컨테이너 로그 형식 파싱
        id: container-parser
```

### Kubernetes 관련 Receivers

```yaml
receivers:
  # Kubernetes 이벤트 수집 (Pod OOMKilled 등)
  k8s_events:
    namespaces: [default, monitoring]

  # 노드/Pod/컨테이너 메트릭 (kubelet API)
  kubeletstats:
    collection_interval: 20s
    auth_type: serviceAccount
    endpoint: "${env:K8S_NODE_NAME}:10250"
    insecure_skip_verify: true
    metric_groups:
      - node
      - pod
      - container
```

---

## 2. Processors

수집한 데이터를 변환/필터링합니다. **파이프라인 순서가 중요합니다.**

### memory_limiter (필수 — 반드시 첫 번째)

Collector의 메모리 사용량을 제한합니다.

```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 400           # 400MB 초과 시 데이터 드랍
    spike_limit_mib: 100
```

### batch (필수 — memory_limiter 다음)

데이터를 모아서 한꺼번에 전송합니다. 네트워크 효율을 높입니다.

```yaml
processors:
  batch:
    timeout: 1s              # 최대 1초 대기
    send_batch_size: 1024    # 1024개 단위로 전송
```

### resource

모든 telemetry에 공통 속성을 추가합니다.

```yaml
processors:
  resource:
    attributes:
      - key: cluster.name
        value: my-eks-cluster
        action: insert
      - key: environment
        value: production
        action: insert
```

### attributes

span/log/metric의 속성을 추가/삭제/변환합니다.

```yaml
processors:
  attributes:
    actions:
      - key: http.user_agent    # 불필요한 속성 삭제 (개인정보 등)
        action: delete
      - key: service.name
        from_attribute: app     # app 속성 값을 service.name으로 복사
        action: insert
```

### k8sattributes (Kubernetes 메타데이터 자동 추가)

Pod에 Kubernetes 메타데이터(namespace, deployment명 등)를 자동으로 붙여줍니다.

```yaml
processors:
  k8sattributes:
    auth_type: serviceAccount
    passthrough: false
    extract:
      metadata:
        - k8s.namespace.name
        - k8s.deployment.name
        - k8s.pod.name
        - k8s.node.name
    pod_association:
      - sources:
          - from: resource_attribute
            name: k8s.pod.ip
```

---

## 3. Exporters

처리된 데이터를 백엔드로 전송합니다.

### OTLP Exporter (Tempo, 다른 Collector로)

```yaml
exporters:
  otlp:
    endpoint: tempo.monitoring:4317
    tls:
      insecure: true

  otlphttp:
    endpoint: http://tempo.monitoring:4318
```

### Prometheus Exporter

Prometheus가 스크레이핑할 수 있도록 메트릭을 HTTP로 노출합니다.

```yaml
exporters:
  prometheus:
    endpoint: 0.0.0.0:8889    # Prometheus가 이 주소를 스크레이핑
```

### Loki Exporter

로그를 Loki로 전송합니다.

```yaml
exporters:
  loki:
    endpoint: http://loki.monitoring:3100/loki/api/v1/push
    default_labels_enabled:
      exporter: false
      job: true
```

### Debug Exporter (개발용)

수집된 데이터를 Collector 로그에 출력합니다. 디버깅용으로만 사용합니다.

```yaml
exporters:
  debug:
    verbosity: detailed
```

---

## 4. 파이프라인 조립

Receivers / Processors / Exporters를 파이프라인으로 연결합니다.

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, k8sattributes, resource, batch]
      exporters: [otlp]      # → Tempo

    metrics:
      receivers: [otlp, prometheus, kubeletstats]
      processors: [memory_limiter, resource, batch]
      exporters: [prometheus]  # → Prometheus scrape

    logs:
      receivers: [otlp, filelog]
      processors: [memory_limiter, k8sattributes, resource, batch]
      exporters: [loki]       # → Loki
```

---

## 5. Collector 배포 모드 비교

| 모드 | 형태 | 용도 |
|------|------|------|
| **DaemonSet** | 노드당 1개 | 노드 로그/메트릭 수집, 앱 사이드카 대체 |
| **Deployment (Gateway)** | 중앙 1~N개 | 앱에서 OTLP 받아 백엔드로 전달, 샘플링 |
| **Sidecar** | Pod당 1개 | 앱과 완전히 격리된 수집 (레거시 환경) |

---

## 참고

- [공식문서 - Collector](https://opentelemetry.io/docs/collector/)
- [공식문서 - Receivers](https://opentelemetry.io/docs/collector/configuration/#receivers)
- [공식문서 - Processors](https://opentelemetry.io/docs/collector/configuration/#processors)
