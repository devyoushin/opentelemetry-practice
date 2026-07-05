# OTel 로그 수집 가이드

OTel Collector로 로그를 수집하고 Loki로 전달합니다. 트레이스와 연동하면 로그에서 Trace로 바로 이동할 수 있습니다.

---

## 로그 수집 흐름

```
[컨테이너 stdout/stderr]
   │ /var/log/pods/.../*.log
   ▼
[OTel Collector — filelog receiver]
   │  파싱, 메타데이터 부착, 트레이스 ID 연결
   ▼
[Loki]
   │
   ▼
[Grafana Explore — 트레이스 클릭 → 관련 로그 자동 조회]
```

---

## 1. Filelog Receiver 설정

컨테이너 로그 파일을 읽어들입니다.

```yaml
receivers:
  filelog:
    include:
      - /var/log/pods/*/*/*.log
    exclude:
      - /var/log/pods/monitoring_*/*/*.log    # monitoring 네임스페이스 제외
    include_file_path: true
    include_file_name: false
    operators:
      # Kubernetes CRI 로그 형식 파싱
      # 형식: 2024-01-10T12:00:00Z stdout F {"level":"info","msg":"..."}
      - type: container
        id: container-parser

      # JSON 로그인 경우 파싱
      - type: json_parser
        id: json-parser
        if: 'body matches "^\\{"'
        parse_from: body
        parse_to: body

      # level 필드 추출 → attribute로 승격
      - type: move
        if: 'body["level"] != nil'
        from: body["level"]
        to: attributes["level"]

      - type: move
        if: 'body["message"] != nil'
        from: body["message"]
        to: body
```

---

## 2. 트레이스 연동 (Trace-Log Correlation)

로그에 `trace_id`와 `span_id`가 있으면 Grafana에서 Trace → Log 연동이 됩니다.

### 앱에서 trace_id를 로그에 포함하는 방법

**Python (structlog + OTel)**

```python
import structlog
from opentelemetry import trace

def add_trace_context(logger, method, event_dict):
    """trace_id, span_id를 로그에 자동 삽입"""
    span = trace.get_current_span()
    if span.is_recording():
        ctx = span.get_span_context()
        event_dict["trace_id"] = format(ctx.trace_id, '032x')
        event_dict["span_id"] = format(ctx.span_id, '016x')
    return event_dict

structlog.configure(
    processors=[
        add_trace_context,
        structlog.processors.JSONRenderer(),
    ]
)

logger = structlog.get_logger()
logger.info("Request processed", path="/api/users", status=200)
# 출력: {"event":"Request processed","path":"/api/users","status":200,"trace_id":"abc123...","span_id":"def456..."}
```

**Java (Logback + OTel)**

```xml
<!-- logback-spring.xml -->
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <!-- OTel Java Agent가 trace_id, span_id를 MDC에 자동 주입 -->
      <provider class="net.logstash.logback.composite.loggingevent.MdcJsonProvider"/>
    </encoder>
  </appender>
</configuration>
```

---

## 3. Loki Exporter 설정

수집한 로그를 Loki로 전송합니다.

```yaml
exporters:
  loki:
    endpoint: http://loki.monitoring:3100/loki/api/v1/push
    default_labels_enabled:
      exporter: false
      job: true
    # OTel resource 속성을 Loki 레이블로 변환
    # 주의: 카디널리티가 낮은 속성만 레이블로 설정
```

### 로그 파이프라인

```yaml
service:
  pipelines:
    logs:
      receivers: [filelog, otlp]
      processors:
        - memory_limiter
        - k8sattributes    # namespace, pod, container 등 자동 추가
        - resource
        - batch
      exporters: [loki]
```

---

## 4. Grafana에서 트레이스 ↔ 로그 연동 설정

### Loki 데이터소스 — Derived Fields 설정

```
Grafana → Configuration → Data Sources → Loki
→ Derived fields 추가:
    Name: TraceID
    Regex: "trace_id":"([a-f0-9]+)"
    Internal link: Tempo 데이터소스 선택
    URL: ${__value.raw}
```

설정 후 Loki 로그에서 `trace_id` 값을 클릭하면 Tempo의 트레이스로 바로 이동합니다.

---

## 5. 로그 수집 상태 확인

```bash
# Collector 로그에서 filelog 수집 상태 확인
kubectl logs -n monitoring daemonset/otelcol-daemonset | grep filelog

# Collector 내부 메트릭으로 수집량 확인
kubectl port-forward -n monitoring daemonset/otelcol-daemonset 8888:8888
curl http://localhost:8888/metrics | grep otelcol_receiver_accepted_log_records

# Loki에서 로그 확인
# Grafana → Explore → Loki → {namespace="default", app="my-app"}
```

---

## 자주 발생하는 문제

| 증상 | 원인 | 해결 |
|------|------|------|
| 로그가 Loki에 없음 | filelog 경로 불일치 | `/var/log/pods` 마운트 확인, `include` 경로 확인 |
| trace_id가 로그에 없음 | 앱에서 trace_id를 로그에 포함 안 함 | SDK로 trace_id를 로그에 주입 |
| 로그 중복 수집 | Promtail과 OTel 동시 사용 | 하나만 선택하거나 제외 규칙 추가 |
| JSON 파싱 안 됨 | 로그 형식이 JSON이 아님 | `json_parser` if 조건 확인 |

---

## 참고

- [공식문서 - Logs](https://opentelemetry.io/docs/concepts/signals/logs/)
- [공식문서 - filelog receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver)
- [공식문서 - Loki exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/lokiexporter)
