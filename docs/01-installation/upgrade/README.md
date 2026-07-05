# OpenTelemetry 업그레이드 가이드

OpenTelemetry 업그레이드는 Operator, Collector image, Collector CRD, Instrumentation CRD가 함께 바뀔 수 있습니다. receiver/processor/exporter 설정 호환성을 먼저 확인합니다.

## 1. 사전 점검

```bash
export NAMESPACE="monitoring"
export RELEASE="opentelemetry-operator"

helm status ${RELEASE} -n ${NAMESPACE}
helm history ${RELEASE} -n ${NAMESPACE}
helm get values ${RELEASE} -n ${NAMESPACE} > values-before-upgrade.yaml
kubectl get opentelemetrycollector,instrumentation -A
kubectl get pods -n ${NAMESPACE}
```

Collector 설정을 백업합니다.

```bash
kubectl get opentelemetrycollector -A -o yaml > otel-collectors-before-upgrade.yaml
kubectl get instrumentation -A -o yaml > instrumentation-before-upgrade.yaml
```

## 2. Helm / Collector 업그레이드

```bash
helm repo update open-telemetry
helm upgrade ${RELEASE} open-telemetry/opentelemetry-operator \
  --namespace ${NAMESPACE} \
  --timeout 10m \
  --wait

kubectl apply -f ../../../ops/config/collector/otelcol-deployment.yaml
kubectl apply -f ../../../ops/config/collector/otelcol-daemonset.yaml
kubectl apply -f ../../../ops/config/collector/instrumentation.yaml
```

## 3. 확인

```bash
kubectl get pods -n ${NAMESPACE}
kubectl get opentelemetrycollector,instrumentation -A
kubectl port-forward -n ${NAMESPACE} deployment/otelcol-deployment 8888:8888
curl http://localhost:8888/metrics | grep otelcol_process_uptime
```

Trace, metric, log exporter가 대상 backend로 전송되는지 확인합니다.

## 4. 롤백

```bash
helm history ${RELEASE} -n ${NAMESPACE}
helm rollback ${RELEASE} <REVISION> -n ${NAMESPACE} --wait
kubectl apply -f otel-collectors-before-upgrade.yaml
kubectl apply -f instrumentation-before-upgrade.yaml
```

CRD schema가 변경된 경우 이전 Operator가 새 필드를 처리하지 못할 수 있습니다.

## 5. systemd / Docker Compose

systemd 설치는 Collector 바이너리와 설정 파일을 백업한 뒤 서비스를 재시작합니다. Docker Compose 설치는 Collector image tag를 변경하고 `docker compose pull && docker compose up -d`를 실행합니다. exporter 대상 backend와 인증 Secret도 함께 점검합니다.

