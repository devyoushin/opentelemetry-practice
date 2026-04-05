# OpenTelemetry 아키텍처

---

## 전체 구조

```
                  ┌─────────────────────────────────────────────────────────────┐
                  │                   EKS Cluster                               │
                  │                                                             │
                  │  [App Pod]                                                  │
                  │  OTel SDK / Agent (자동 또는 수동 계측)                     │
                  │     │ OTLP (gRPC:4317 / HTTP:4318)                         │
                  │     ▼                                                       │
                  │  [OTel Collector — Deployment (Gateway)]                   │
                  │  Receiver → Processor → Exporter                           │
                  │     │           │            │                              │
                  │     │     k8sattributes  batch                              │
                  │     │     memory_limiter resource                           │
                  │     │                                                       │
                  │     ├── traces  ──────────────────────▶ [Grafana Tempo]    │
                  │     ├── metrics ──────────────────────▶ [Prometheus]       │
                  │     └── logs    ──────────────────────▶ [Loki]             │
                  │                                              │              │
                  │  [OTel Collector — DaemonSet]               │              │
                  │  (노드 로그 + kubelet 메트릭 수집)            │              │
                  │     │                                        ▼              │
                  │     └── logs/metrics ──────▶ Gateway ─▶ Prometheus/Loki   │
                  │                                                             │
                  │  [OTel Operator]                                            │
                  │  Instrumentation CR → init container 자동 주입              │
                  └─────────────────────────────────────────────────────────────┘
                                              │
                              ┌───────────────┼───────────────┐
                              ▼               ▼               ▼
                        Grafana Tempo    Prometheus         Loki
                        (트레이스)       (메트릭)           (로그)
                              │               │               │
                              └───────────────┼───────────────┘
                                              ▼
                                    [Grafana Dashboard]
                                    Explore / Correlate
```

---

## 컴포넌트 상세

### OTel SDK (앱 레이어)

앱 코드에서 telemetry 데이터를 생성하는 라이브러리입니다.

- **자동 계측 (Auto)**: OTel Operator가 init container로 언어별 에이전트 주입
  - HTTP, DB, Redis, gRPC 등 주요 라이브러리 자동 계측
  - 코드 수정 없이 적용
- **수동 계측 (Manual)**: SDK API로 커스텀 Span / Metric / Log 추가
  - 비즈니스 로직, 큐 처리 등 프레임워크가 커버 못하는 영역

```
OTel SDK
  │ 생성
  ├── Trace  → Span (작업 단위)
  ├── Metric → Counter / Gauge / Histogram
  └── Log    → Structured log (trace_id 자동 연결)
```

---

### OTel Collector

앱과 백엔드 사이의 **telemetry 데이터 파이프라인**입니다.

```
[Receivers]  →  [Processors]  →  [Exporters]
  수신             처리/변환        전송
```

| 역할 | 설명 |
|------|------|
| **Receiver** | 앱 SDK, 다른 Collector, 파일에서 데이터 수신 |
| **Processor** | 필터링, 속성 추가/삭제, 샘플링, 배치 처리 |
| **Exporter** | 백엔드(Tempo, Prometheus, Loki)로 전송 |

#### 배포 모드 비교

| 모드 | 형태 | 주 용도 |
|------|------|---------|
| **Deployment (Gateway)** | 중앙 1~N개 Pod | 앱 OTLP 수신, Tail Sampling, 백엔드 전달 |
| **DaemonSet** | 노드당 1개 Pod | 노드 로그 수집 (filelog), kubelet 메트릭 |
| **Sidecar** | Pod당 1개 컨테이너 | 레거시 환경, 격리 필요 시 (EKS에서는 잘 사용 안 함) |

> **EKS 권장 구성**: DaemonSet(노드 로그/메트릭) + Deployment Gateway(앱 트레이스) 조합

---

### OTel Operator

Kubernetes Operator로, CRD를 통해 Collector와 Auto Instrumentation을 관리합니다.

| CRD | 역할 |
|-----|------|
| `OpenTelemetryCollector` | Collector 배포 (Deployment/DaemonSet/Sidecar 모드) |
| `Instrumentation` | 언어별 에이전트 자동 주입 설정 |

```bash
# 설치된 CRD 확인
kubectl get crd | grep opentelemetry
# opentelemetrycollectors.opentelemetry.io
# instrumentations.opentelemetry.io
```

---

## 세 가지 Signal의 흐름

### Traces (분산 트레이싱)

```
[App Pod]
OTel SDK → Span 생성 (trace_id, span_id, parent_span_id)
   │ OTLP gRPC
   ▼
[Collector Gateway]
k8sattributes → k8s.namespace, k8s.pod.name, k8s.deployment.name 추가
batch → 배치 전송
   │ OTLP
   ▼
[Grafana Tempo]
TraceQL로 조회: { resource.service.name = "my-app" && duration > 1s }
   │
   ▼
[Grafana Explore]
Span 계층 구조 시각화 → "Logs for this span" → Loki 연동
```

### Metrics (메트릭)

```
[App Pod]
OTel SDK → Counter / Gauge / Histogram 측정
   │ OTLP gRPC
   ▼
[Collector Gateway]
Prometheus Exporter → :8889 HTTP 엔드포인트 노출
   │ Prometheus scrape
   ▼
[Prometheus]
PromQL로 조회: rate(http_requests_total[5m])
   │
   ▼
[Grafana Dashboard / Alert]

──────────────────────────────────────
[DaemonSet Collector] (병행)
kubeletstats receiver → 노드/Pod CPU, 메모리, 네트워크
   │ OTLP → Gateway → Prometheus
```

### Logs (로그)

```
[App Pod]
stdout JSON 로그 (trace_id 포함)
   │ /var/log/pods/...
   ▼
[DaemonSet Collector]
filelog receiver → 파싱 + trace_id 추출
k8sattributes → namespace, pod명 추가
   │ OTLP → Gateway
   ▼
[Collector Gateway]
Loki Exporter → push
   │
   ▼
[Loki]
LogQL: {k8s_namespace_name="default"} | trace_id="abc123"
   │
   ▼
[Grafana Explore]
Derived Fields → trace_id 클릭 → Tempo로 이동
```

---

## Trace - Metric - Log 상관관계 (Correlation)

OTel의 핵심 가치는 세 Signal을 **trace_id로 연결**하는 것입니다.

```
Grafana Tempo에서 느린 트레이스 발견
  │ "Logs for this span" 클릭
  ▼
Loki에서 해당 trace_id 로그 조회
  │ 에러 메시지 확인
  ▼
Prometheus에서 해당 시간대 메트릭 조회
  │ CPU 스파이크, DB 커넥션 풀 고갈 등 원인 파악
  ▼
근본 원인 파악 → 수정
```

| Signal | 역할 | 도구 |
|--------|------|------|
| **Traces** | "어디서 느린가?" — 요청 흐름 추적 | Tempo + TraceQL |
| **Metrics** | "얼마나 자주, 얼마나 느린가?" — 집계 수치 | Prometheus + PromQL |
| **Logs** | "무슨 일이 있었나?" — 상세 이벤트 | Loki + LogQL |

---

## OTel vs 기존 도구

| 기능 | 기존 방식 | OTel 방식 |
|------|----------|-----------|
| 트레이싱 | Jaeger SDK, Zipkin SDK | OTel SDK → 어떤 백엔드도 연결 가능 |
| 메트릭 | Prometheus client library | OTel SDK → Prometheus / Datadog / Cloud |
| 로그 | Fluentd / Fluent Bit | OTel Collector filelog receiver |
| 계측 변경 | SDK 교체 → 코드 전면 수정 | 백엔드만 교체, 코드 무변경 |

> OTel은 **계측과 백엔드를 분리**합니다. 오늘 Jaeger를 쓰다가 Tempo로 바꿔도
> 앱 코드 수정 없이 Collector 설정만 변경하면 됩니다.

---

## 참고 링크

- [OpenTelemetry 공식 문서](https://opentelemetry.io/docs/)
- [OTel Collector 아키텍처](https://opentelemetry.io/docs/collector/architecture/)
- [OTel Operator (Kubernetes)](https://opentelemetry.io/docs/kubernetes/operator/)
- [Grafana Tempo 공식 문서](https://grafana.com/docs/tempo/latest/)
