# OpenTelemetry on EKS — 실습 저장소

EKS 환경에서 OpenTelemetry를 처음부터 운영 수준까지 학습하는 실습 저장소입니다.

---

## 환경 정보

| 항목 | 값 |
|---|---|
| 플랫폼 | AWS EKS |
| OTel Operator Helm Chart | `open-telemetry/opentelemetry-operator` (0.100.x) |
| OTel Collector | `otel/opentelemetry-collector-contrib` (0.100.x) |
| 네임스페이스 | `monitoring` (OTel 스택), `default` (앱) |
| 백엔드 | Grafana Tempo (트레이스), Prometheus (메트릭), Loki (로그) |

---

## 사전 요구사항

```bash
# 필요 도구 확인
kubectl version --client      # >= 1.25
helm version                  # >= 3.10
aws --version                 # AWS CLI v2

# EKS 클러스터 접속 확인
kubectl get nodes

# 백엔드 서비스 확인 (Tempo, Prometheus, Loki, Grafana)
kubectl get svc -n monitoring
```

> 백엔드(Tempo, Prometheus, Loki, Grafana)가 `monitoring` 네임스페이스에 설치되어 있어야 합니다.

---

## 빠른 시작 (Quick Start)

```bash
# 1. Helm 저장소 추가
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo add jetstack https://charts.jetstack.io
helm repo update

# 2. cert-manager 설치 (OTel Operator 필수)
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true

# 3. OTel Operator 설치
kubectl create namespace monitoring
helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace monitoring \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-contrib"

# 4. Collector 배포 (Gateway + DaemonSet)
kubectl apply -f collector/otelcol-deployment.yaml
kubectl apply -f collector/otelcol-daemonset.yaml

# 5. Auto Instrumentation CR 생성
kubectl apply -f collector/instrumentation.yaml

# 6. 샘플 앱 배포
kubectl apply -f app/deployment.yaml
kubectl apply -f app/service.yaml

# 7. 상태 확인
kubectl get pods -n monitoring
```

---

## 학습 경로

### 1단계: 설치
- [Helm으로 OTel Operator + Collector 설치](./install.md)

### 2단계: 핵심 개념
- [아키텍처 개요](./architecture-guide.md)
- [Collector 파이프라인 — Receiver / Processor / Exporter](./collector-guide.md)
- [분산 트레이싱 — Trace, Span, Context Propagation](./tracing-guide.md)

### 3단계: Signal 심화
- [메트릭 수집 — Counter/Gauge/Histogram, Prometheus 연동](./metrics-guide.md)
- [로그 수집 — Filelog Receiver, 트레이스-로그 연동](./logging-guide.md)

### 4단계: 계측 & 샘플링
- [계측 — 자동(Auto) / 수동(Manual) 계측, OTel Operator](./instrumentation-guide.md)
- [샘플링 — Head / Tail Sampling 전략](./sampling-guide.md)

### 5단계: 백엔드 연동
- [백엔드 연동 — Tempo + Prometheus + Loki + Grafana](./backend-integration-guide.md)

### 6단계: 문제 해결
- [트러블슈팅 가이드](./troubleshooting-guide.md)

### 실습
- [End-to-End 실습 (앱 배포 → 트레이스 → 메트릭 → 로그 연동)](./otel-demo-test.md)

---

## 저장소 구조

```
opentelemetry-practice/
├── README.md
├── install.md                     # OTel Operator + Collector 설치
├── architecture-guide.md          # 전체 아키텍처 개요
├── collector-guide.md             # Collector 파이프라인
├── tracing-guide.md               # 분산 트레이싱 개념
├── metrics-guide.md               # 메트릭 수집
├── logging-guide.md               # 로그 수집
├── instrumentation-guide.md       # 자동/수동 계측
├── sampling-guide.md              # 샘플링 전략
├── backend-integration-guide.md   # 백엔드 연동
├── troubleshooting-guide.md       # 트러블슈팅
├── otel-demo-test.md              # End-to-End 실습
├── collector/
│   ├── otelcol-deployment.yaml    # Gateway Collector (Deployment)
│   ├── otelcol-daemonset.yaml     # Node Collector (DaemonSet)
│   └── instrumentation.yaml       # Auto Instrumentation CR
└── app/
    ├── deployment.yaml            # 계측된 샘플 앱 (Python)
    └── service.yaml               # 샘플 앱 Service
```

---

## 아키텍처 요약

```
[App Pod]
OTel SDK / Agent (자동 계측 — init container 주입)
   │ OTLP (gRPC:4317)
   ▼
[OTel Collector — Deployment (Gateway)]
Receiver → k8sattributes → resource → batch → Exporter
   │
   ├── traces  ──────────────────▶ [Grafana Tempo]
   ├── metrics ──────────────────▶ [Prometheus]
   └── logs    ──────────────────▶ [Loki]

[OTel Collector — DaemonSet]
filelog(노드 로그) + kubeletstats(노드 메트릭)
   └── → Gateway → Prometheus / Loki

[OTel Operator]
Instrumentation CR → init container 자동 주입 관리
```

| 컴포넌트 | 역할 |
|---|---|
| **OTel SDK / Agent** | 앱에서 Trace / Metric / Log 생성 및 OTLP 전송 |
| **Collector (Gateway)** | OTLP 수신, 속성 보강, 백엔드로 라우팅 |
| **Collector (DaemonSet)** | 노드별 로그 수집 (filelog), kubelet 메트릭 수집 |
| **OTel Operator** | Collector CR 관리, 언어별 에이전트 자동 주입 |
| **Grafana Tempo** | 트레이스 저장 및 TraceQL 조회 |
| **Prometheus** | 메트릭 저장 및 PromQL 조회, 알림 |
| **Loki** | 로그 저장 및 LogQL 조회, trace_id 연동 |

---

## 참고 링크

- [OpenTelemetry 공식 문서](https://opentelemetry.io/docs/)
- [OTel Collector 공식 문서](https://opentelemetry.io/docs/collector/)
- [OTel Operator (Kubernetes)](https://opentelemetry.io/docs/kubernetes/operator/)
- [opentelemetry-helm-charts](https://github.com/open-telemetry/opentelemetry-helm-charts)
- [Grafana Tempo 공식 문서](https://grafana.com/docs/tempo/latest/)
