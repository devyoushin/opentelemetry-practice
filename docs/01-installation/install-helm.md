# OpenTelemetry 설치 (Helm)

EKS 환경에서 OTel Operator와 Collector를 설치한다.

## 사전 준비

```bash
kubectl get nodes
helm version
aws sts get-caller-identity
kubectl get svc -n monitoring
```

## 설치 순서

1. `cert-manager` 설치
2. `opentelemetry-operator` 설치
3. `otelcol-deployment` 설치
4. `otelcol-daemonset` 설치
5. `instrumentation` CR 적용

## 확인

```bash
kubectl get pods -n monitoring
kubectl get crd | grep opentelemetry
kubectl port-forward -n monitoring deployment/otelcol-deployment 8888:8888
curl http://localhost:8888/metrics | grep otelcol_process_uptime
```

## 업그레이드 / 롤백 / 삭제

```bash
helm upgrade opentelemetry-operator open-telemetry/opentelemetry-operator -n monitoring
kubectl apply -f ../../ops/config/collector/otelcol-deployment.yaml
kubectl apply -f ../../ops/config/collector/otelcol-daemonset.yaml
helm rollback opentelemetry-operator 1 -n monitoring
helm uninstall opentelemetry-operator -n monitoring
```
