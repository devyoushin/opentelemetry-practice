# 계측(Instrumentation) 가이드

앱에 OTel 계측을 추가하는 두 가지 방식을 다룹니다.

---

## 자동 계측 vs 수동 계측

| 방식 | 장점 | 단점 |
|------|------|------|
| **자동 계측 (Auto)** | 코드 수정 없음, 빠른 적용 | 커스텀 Span 추가 불가, 언어 제한 |
| **수동 계측 (Manual)** | 세밀한 제어, 비즈니스 로직 Span 추가 | 코드 수정 필요 |

> 실제로는 **자동 계측으로 시작하고, 필요한 부분만 수동으로 추가**하는 방식을 권장합니다.

---

## 1. 자동 계측 — OTel Operator 사용 (권장)

OTel Operator의 `Instrumentation` CR을 사용하면 Pod에 언어별 에이전트를 자동으로 주입합니다.

### Instrumentation CR 생성

```yaml
# collector/instrumentation.yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: my-instrumentation
  namespace: default
spec:
  exporter:
    endpoint: http://otelcol-deployment.monitoring:4317   # Collector gRPC 주소

  propagators:
    - tracecontext    # W3C TraceContext (표준)
    - baggage

  sampler:
    type: parentbased_traceidratio
    argument: "1.0"   # 100% 샘플링 (운영에서는 낮춤)

  # 언어별 에이전트 설정
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
    env:
      - name: OTEL_INSTRUMENTATION_JDBC_ENABLED
        value: "true"

  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest

  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:latest

  go:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-go:latest
```

```bash
kubectl apply -f collector/instrumentation.yaml
```

### Pod에 자동 계측 활성화

Deployment에 어노테이션 한 줄만 추가합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    metadata:
      annotations:
        # 언어에 맞는 어노테이션 선택
        instrumentation.opentelemetry.io/inject-java: "my-instrumentation"
        # instrumentation.opentelemetry.io/inject-python: "my-instrumentation"
        # instrumentation.opentelemetry.io/inject-nodejs: "my-instrumentation"
        # instrumentation.opentelemetry.io/inject-go: "my-instrumentation"
```

```bash
kubectl apply -f app/deployment.yaml

# 에이전트가 init container로 주입되었는지 확인
kubectl describe pod -l app=my-app | grep -A5 "Init Containers"
# opentelemetry-auto-instrumentation 컨테이너가 보여야 함
```

---

## 2. 자동 계측 — 언어별 에이전트 직접 사용

OTel Operator 없이 에이전트를 직접 추가합니다.

### Java

```dockerfile
# Dockerfile
FROM eclipse-temurin:17
ADD https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar /app/
ENV JAVA_TOOL_OPTIONS="-javaagent:/app/opentelemetry-javaagent.jar"
ENV OTEL_EXPORTER_OTLP_ENDPOINT="http://otelcol-deployment.monitoring:4317"
ENV OTEL_SERVICE_NAME="my-java-app"
```

### Python

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap --action=install

# 실행 시
opentelemetry-instrument \
  --exporter_otlp_endpoint=http://otelcol-deployment.monitoring:4317 \
  --service_name=my-python-app \
  python app.py
```

### Node.js

```bash
npm install @opentelemetry/auto-instrumentations-node \
            @opentelemetry/exporter-trace-otlp-grpc

# 실행 시 (--require로 로드)
node --require @opentelemetry/auto-instrumentations-node/register app.js
```

환경변수:
```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://otelcol-deployment.monitoring:4317
OTEL_SERVICE_NAME=my-node-app
OTEL_TRACES_EXPORTER=otlp
```

---

## 3. 수동 계측 — 커스텀 Span 추가

자동 계측이 커버하지 못하는 비즈니스 로직에 수동으로 Span을 추가합니다.

### Python

```python
from opentelemetry import trace

tracer = trace.get_tracer("my-app")

def process_order(order_id: str):
    # 커스텀 Span 시작
    with tracer.start_as_current_span("process_order") as span:
        span.set_attribute("order.id", order_id)

        # 중첩 Span
        with tracer.start_as_current_span("validate_payment"):
            result = validate_payment(order_id)
            if not result.success:
                span.set_status(trace.Status(trace.StatusCode.ERROR, "Payment failed"))
                span.record_exception(PaymentException("Payment failed"))
                raise PaymentException("Payment failed")

        span.set_attribute("order.status", "processed")
```

### Go

```go
import "go.opentelemetry.io/otel"

tracer := otel.Tracer("my-app")

func processOrder(ctx context.Context, orderID string) error {
    ctx, span := tracer.Start(ctx, "process_order",
        trace.WithAttributes(attribute.String("order.id", orderID)),
    )
    defer span.End()

    if err := validatePayment(ctx, orderID); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return err
    }

    span.SetAttributes(attribute.String("order.status", "processed"))
    return nil
}
```

---

## 4. 계측 확인

```bash
# Collector가 트레이스를 받고 있는지 확인
kubectl port-forward -n monitoring deployment/otelcol-deployment 8888:8888
curl http://localhost:8888/metrics | grep otelcol_receiver_accepted_spans

# Grafana Tempo에서 트레이스 확인
kubectl port-forward svc/grafana -n monitoring 3000:80
# Explore → Tempo → service.name으로 검색
```

---

## 참고

- [공식문서 - Instrumentation](https://opentelemetry.io/docs/concepts/instrumentation/)
- [공식문서 - OTel Operator Auto Instrumentation](https://opentelemetry.io/docs/kubernetes/operator/automatic/)
- [Java 자동 계측 지원 라이브러리 목록](https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/docs/supported-libraries.md)
