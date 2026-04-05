# Helm으로 OpenTelemetry 설치

EKS 환경에서 OTel Operator + Collector를 설치합니다.
Operator는 Auto Instrumentation과 OpenTelemetryCollector CR 관리에 필요합니다.

---

## 사전 조건 확인

```bash
# kubectl 연결 확인
kubectl get nodes

# Helm 버전 확인 (>= 3.10 권장)
helm version

# AWS CLI 설정 확인
aws sts get-caller-identity

# 백엔드 서비스 확인 (Tempo, Prometheus, Loki, Grafana)
kubectl get svc -n monitoring
# tempo, prometheus-server, loki, grafana 가 보여야 함
```

---

## 1. 네임스페이스 생성

```bash
kubectl create namespace monitoring
```

---

## 2. Helm 저장소 추가

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo add jetstack https://charts.jetstack.io   # cert-manager (Operator 필수)
helm repo update

# 사용 가능한 버전 확인
helm search repo open-telemetry/opentelemetry-operator --versions | head -5
```

---

## 3. cert-manager 설치 (OTel Operator 필수)

OTel Operator의 웹훅(Webhook)은 TLS 인증서가 필요합니다.

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

설치 확인:
```bash
kubectl get pods -n cert-manager
# cert-manager-xxx, cert-manager-cainjector-xxx, cert-manager-webhook-xxx 가 Running인지 확인
```

> **팁**: cert-manager가 Ready가 될 때까지 약 30~60초 대기 후 다음 단계를 진행하세요.
> Operator 설치가 너무 빠르면 Webhook TLS 에러가 발생할 수 있습니다.

---

## 4. OTel Operator 설치

Auto Instrumentation CR과 OpenTelemetryCollector CR을 관리합니다.

```bash
helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace monitoring \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-contrib"
```

설치 확인:
```bash
kubectl get pods -n monitoring
# opentelemetry-operator-xxx 가 Running인지 확인

# CRD 확인
kubectl get crd | grep opentelemetry
# opentelemetrycollectors.opentelemetry.io
# instrumentations.opentelemetry.io
```

---

## 5-A. OTel Collector 설치 — Deployment (Gateway)

앱에서 OTLP로 트레이스/메트릭을 받아 백엔드로 전달하는 중앙 집중 방식입니다.

```bash
kubectl apply -f collector/otelcol-deployment.yaml
```

설치 확인:
```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=otelcol-deployment
# otelcol-deployment-xxx 가 Running인지 확인

kubectl get svc -n monitoring -l app.kubernetes.io/name=otelcol-deployment
# otelcol-deployment   ClusterIP   10.100.x.x   4317/TCP,4318/TCP,8888/TCP
```

---

## 5-B. OTel Collector 설치 — DaemonSet

각 노드에서 컨테이너 로그(filelog)와 kubelet 메트릭을 수집합니다.

```bash
kubectl apply -f collector/otelcol-daemonset.yaml
```

설치 확인:
```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=otelcol-daemonset -o wide
# 노드 수만큼 Pod가 생성되어야 함 (DESIRED = 노드 수)

kubectl get daemonset -n monitoring
# NAME               DESIRED   CURRENT   READY
# otelcol-daemonset  3         3         3
```

---

## 6. Auto Instrumentation CR 생성

언어별 에이전트 자동 주입 설정을 생성합니다.

```bash
kubectl apply -f collector/instrumentation.yaml

# 생성 확인
kubectl get instrumentation -n default
# NAME                 AGE
# my-instrumentation   5s
```

---

## 7. 전체 설치 상태 확인

```bash
kubectl get all -n monitoring
```

정상 설치 시 예상 출력:
```
NAME                                                READY   STATUS    RESTARTS
pod/opentelemetry-operator-xxx                      2/2     Running   0
pod/otelcol-deployment-xxx                          1/1     Running   0
pod/otelcol-daemonset-node1                         1/1     Running   0
pod/otelcol-daemonset-node2                         1/1     Running   0
pod/otelcol-daemonset-node3                         1/1     Running   0

NAME                         TYPE        CLUSTER-IP
service/otelcol-deployment   ClusterIP   10.100.x.x

NAME                              DESIRED   CURRENT   READY
daemonset.apps/otelcol-daemonset  3         3         3
```

---

## 8. Collector 헬스 체크

```bash
# Gateway Collector 내부 메트릭 포트 포워드
kubectl port-forward -n monitoring deployment/otelcol-deployment 8888:8888 &

# Collector 상태 확인
curl http://localhost:8888/metrics | grep otelcol_process_uptime

# 수신/전송 현황 확인
curl -s http://localhost:8888/metrics | grep -E \
  "otelcol_(receiver_accepted|exporter_sent)_(spans|metric_points|log_records)"
```

---

## 업그레이드

```bash
# 현재 설치된 버전 확인
helm list -n monitoring

# Operator 업그레이드
helm upgrade opentelemetry-operator open-telemetry/opentelemetry-operator \
  --namespace monitoring \
  --set "manager.collectorImage.repository=otel/opentelemetry-collector-contrib"

# Collector는 CR 재적용 (Operator가 관리)
kubectl apply -f collector/otelcol-deployment.yaml
kubectl apply -f collector/otelcol-daemonset.yaml

# 업그레이드 히스토리 확인
helm history opentelemetry-operator -n monitoring
```

---

## 롤백

```bash
# 이전 버전으로 롤백
helm rollback opentelemetry-operator 1 -n monitoring
```

---

## 삭제

```bash
# Collector CR 삭제 (Operator가 관련 리소스 정리)
kubectl delete -f collector/otelcol-deployment.yaml
kubectl delete -f collector/otelcol-daemonset.yaml
kubectl delete -f collector/instrumentation.yaml

# OTel Operator 삭제
helm uninstall opentelemetry-operator -n monitoring

# cert-manager 삭제
helm uninstall cert-manager -n cert-manager
kubectl delete namespace cert-manager

# 네임스페이스 삭제
kubectl delete namespace monitoring
```

---

## 설치 시 자주 발생하는 문제

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| Operator `CrashLoopBackOff` | cert-manager 미설치 또는 아직 준비 안 됨 | `kubectl get pods -n cert-manager` 확인 후 재시도 |
| Operator Webhook 에러 | cert-manager 인증서 발급 지연 | 60초 대기 후 `helm upgrade` 재실행 |
| Collector `Pending` | 리소스 부족 | 노드 용량 확인, requests/limits 조정 |
| `CRD not found` 에러 | Operator 설치 전 CR 적용 시도 | Operator Pod Running 확인 후 CR 재적용 |
| DaemonSet Pod 수 부족 | 노드 Taint / Toleration 미설정 | DaemonSet에 tolerations 추가 |
| OTLP gRPC 연결 실패 | 서비스 이름/네임스페이스 오류 | `kubectl get svc -n monitoring` 확인 |
