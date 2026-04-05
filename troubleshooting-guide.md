# 트러블슈팅 가이드

---

## 진단 흐름

```
트레이스/메트릭/로그가 안 보인다
  │
  ├── 1. Operator / Collector 파드 상태 확인
  ├── 2. Collector 로그 확인 (에러 메시지)
  ├── 3. Collector 내부 메트릭 확인 (:8888)
  ├── 4. 앱 계측 확인 (init container 주입 여부)
  └── 5. 백엔드 연결 확인 (Tempo / Prometheus / Loki)
```

---

## 빠른 상태 점검

```bash
# 전체 파드 상태 한 번에 확인
kubectl get pods -n monitoring

# Collector 로그 (에러 위주로 확인)
kubectl logs -n monitoring deployment/otelcol-deployment --tail=50 | grep -E "error|Error|WARN"

# Collector 내부 메트릭 (수신/전송 현황)
kubectl port-forward -n monitoring deployment/otelcol-deployment 8888:8888 &
curl -s http://localhost:8888/metrics | grep -E "otelcol_(receiver|exporter)"
```

---

## 1. 트레이스가 Tempo에 없다

### 원인 1: Auto Instrumentation이 적용되지 않음

```bash
# init container 주입 여부 확인
kubectl describe pod -l app=my-app | grep -A10 "Init Containers"
# "opentelemetry-auto-instrumentation" 이 없으면 미주입

# Instrumentation CR 존재 여부 확인
kubectl get instrumentation -n default
# NAME                 AGE
# my-instrumentation   5m  ← 이 줄이 없으면 CR 미생성

# 어노테이션 확인
kubectl get pod -l app=my-app -o jsonpath='{.items[0].metadata.annotations}'
# instrumentation.opentelemetry.io/inject-python 이 있어야 함
```

**해결**: Instrumentation CR 생성 후 Pod 재시작
```bash
kubectl apply -f collector/instrumentation.yaml
kubectl rollout restart deployment/my-app
```

---

### 원인 2: OTLP 엔드포인트 주소 오류

```bash
# 환경변수 확인 (앱 Pod 내부)
kubectl exec -it $(kubectl get pod -l app=my-app -o name | head -1) -- env | grep OTEL
# OTEL_EXPORTER_OTLP_ENDPOINT=http://otelcol-deployment.monitoring:4317  ← 이 값 확인

# Collector 서비스 이름 확인
kubectl get svc -n monitoring | grep otelcol
# otelcol-deployment   ClusterIP   10.100.x.x   4317/TCP,4318/TCP
```

**해결**: `Instrumentation` CR의 `exporter.endpoint` 값이 Collector 서비스와 일치하는지 확인
```yaml
spec:
  exporter:
    endpoint: http://otelcol-deployment.monitoring:4317  # 서비스명.네임스페이스:포트
```

---

### 원인 3: Collector → Tempo 연결 실패

```bash
# Collector 로그에서 Tempo 관련 에러 확인
kubectl logs -n monitoring deployment/otelcol-deployment | grep -i tempo

# Tempo 서비스 확인
kubectl get svc -n monitoring | grep tempo
# tempo   ClusterIP   10.100.x.x   3200/TCP,4317/TCP,4318/TCP

# Collector에서 Tempo로 TCP 연결 테스트
kubectl exec -n monitoring deployment/otelcol-deployment -- nc -zv tempo.monitoring 4317
```

**해결**: Collector exporter의 `endpoint` 값 수정
```yaml
exporters:
  otlp:
    endpoint: tempo.monitoring:4317  # http:// 없이 gRPC 주소
    tls:
      insecure: true
```

---

### 원인 4: Collector가 스팬을 드랍하고 있음

```bash
# 드랍된 스팬 수 확인
curl -s http://localhost:8888/metrics | grep otelcol_processor_dropped_spans
# 값이 계속 올라가면 memory_limiter가 트리거되는 중

# 메모리 사용량 확인
curl -s http://localhost:8888/metrics | grep otelcol_process_memory_rss
```

**해결**: Collector 리소스 제한 또는 `memory_limiter` 설정 조정
```yaml
processors:
  memory_limiter:
    limit_mib: 800    # 기본 400에서 증가
    spike_limit_mib: 200
```

---

## 2. 메트릭이 Prometheus에 없다

### 원인 1: Prometheus가 Collector를 스크레이핑하지 않음

```bash
# Collector Prometheus 엔드포인트 직접 확인
kubectl port-forward -n monitoring deployment/otelcol-deployment 8889:8889 &
curl -s http://localhost:8889/metrics | head -20
# 값이 나오면 Collector는 정상 → Prometheus 설정 문제

# Prometheus에서 타겟 확인
kubectl port-forward -n monitoring svc/prometheus 9090:9090 &
# http://localhost:9090/targets 에서 otelcol 타겟 확인
```

**해결**: Prometheus에 Collector 스크레이핑 설정 추가
```yaml
# Prometheus scrape config
- job_name: otel-collector
  static_configs:
    - targets: ['otelcol-deployment.monitoring:8889']
```

---

### 원인 2: 메트릭 이름이 변환됨 (OTel → Prometheus 형식)

OTel SDK의 메트릭 이름과 Prometheus에서 조회되는 이름이 다를 수 있습니다.

```bash
# 실제 메트릭 이름 확인
curl -s http://localhost:8889/metrics | grep http_
# http_server_duration_milliseconds_bucket
# ↑ OTel의 http.server.duration 이 변환된 형식
```

> OTel 메트릭 이름의 `.`은 Prometheus에서 `_`로 변환됩니다.
> 예: `http.server.request_count` → `http_server_request_count_total`

---

### 원인 3: kubeletstats receiver 권한 부족

```bash
# DaemonSet Collector 로그에서 403/권한 에러 확인
kubectl logs -n monitoring daemonset/otelcol-daemonset | grep -i "forbidden\|403\|unauthorized"
```

**해결**: ClusterRole에 `nodes/stats` 권한 추가
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
rules:
  - apiGroups: [""]
    resources: ["nodes/stats", "nodes/proxy"]
    verbs: ["get", "list", "watch"]
```

---

## 3. 로그가 Loki에 없다

### 원인 1: DaemonSet Collector가 로그 파일을 찾지 못함

```bash
# DaemonSet 파드가 노드마다 떠 있는지 확인
kubectl get pods -n monitoring -o wide | grep daemonset
# 노드 수만큼 파드가 있어야 함

# DaemonSet Collector 로그 확인
kubectl logs -n monitoring daemonset/otelcol-daemonset | grep -i "filelog\|error"

# 볼륨 마운트 확인
kubectl describe pod -n monitoring -l app.kubernetes.io/name=otelcol-daemonset | grep -A5 "Volumes"
# /var/log/pods 가 마운트되어 있어야 함
```

---

### 원인 2: 로그가 JSON 형식이 아님

```bash
# 앱 로그 원본 확인
kubectl logs -l app=my-app --tail=5
# {"timestamp": "...", "message": "...", "trace_id": "..."}  ← JSON이어야 함
# 일반 텍스트라면 filelog의 operators 파이프라인 수정 필요
```

---

### 원인 3: Loki 연결 실패

```bash
# Collector 로그에서 Loki 에러 확인
kubectl logs -n monitoring deployment/otelcol-deployment | grep -i loki

# Loki 서비스 확인
kubectl get svc -n monitoring | grep loki

# Loki push API 직접 테스트
kubectl exec -n monitoring deployment/otelcol-deployment -- \
  curl -s -o /dev/null -w "%{http_code}" \
  http://loki.monitoring:3100/ready
# 200이어야 함
```

---

### 원인 4: 트레이스-로그 연동이 안 됨

Tempo에서 "Logs for this span"이 동작하지 않는 경우입니다.

```bash
# 앱 로그에 trace_id가 포함되어 있는지 확인
kubectl logs -l app=my-app --tail=5 | jq '.trace_id'
# null 이면 앱 코드에서 trace_id를 로그에 넣지 않는 중
```

**해결**: Grafana Loki 데이터소스에 Derived Fields 설정
```
Grafana → Configuration → Data Sources → Loki
→ Derived Fields 탭
→ Name: TraceID
→ Regex: "trace_id":"([a-f0-9]+)"
→ URL: http://<tempo>:3200/tempo/api/traces/${__value.raw}
→ Internal link: Tempo 데이터소스 선택
```

---

## 4. OTel Operator 관련

### Operator Pod가 Running이 아님

```bash
# Operator 상태 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=opentelemetry-operator

# 로그 확인
kubectl logs -n monitoring -l app.kubernetes.io/name=opentelemetry-operator
```

**원인 1: cert-manager 미설치**
```bash
kubectl get pods -n cert-manager
# cert-manager-xxx 파드가 없으면 cert-manager 먼저 설치 필요
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true
```

**원인 2: Webhook TLS 인증서 발급 대기**
```bash
# cert-manager가 인증서를 발급할 시간 필요 (보통 30초~1분)
kubectl get certificate -n monitoring
# READY: True 가 될 때까지 대기
```

---

### Auto Instrumentation이 적용되었는데 트레이스가 안 나옴

```bash
# 언어별 에이전트 버전 확인 (Instrumentation CR)
kubectl get instrumentation my-instrumentation -n default -o yaml | grep image

# init container 실행 로그 확인
kubectl logs $(kubectl get pod -l app=my-app -o name | head -1) \
  -c opentelemetry-auto-instrumentation

# Go의 경우 eBPF 사용 → 추가 권한 필요
kubectl get instrumentation my-instrumentation -n default -o yaml | grep go
```

---

## 5. 공통 진단 명령어

```bash
# ── Operator ──────────────────────────────────────────────────
kubectl get pods -n monitoring -l app.kubernetes.io/name=opentelemetry-operator
kubectl logs -n monitoring -l app.kubernetes.io/name=opentelemetry-operator --tail=30

# ── Collector (Gateway) ───────────────────────────────────────
kubectl get pods -n monitoring -l app.kubernetes.io/name=otelcol-deployment
kubectl logs -n monitoring deployment/otelcol-deployment --tail=50
kubectl describe pod -n monitoring -l app.kubernetes.io/name=otelcol-deployment

# ── Collector (DaemonSet) ─────────────────────────────────────
kubectl get pods -n monitoring -l app.kubernetes.io/name=otelcol-daemonset -o wide
kubectl logs -n monitoring daemonset/otelcol-daemonset --tail=30

# ── Collector 내부 메트릭 ─────────────────────────────────────
kubectl port-forward -n monitoring deployment/otelcol-deployment 8888:8888 &
curl -s http://localhost:8888/metrics | grep -E \
  "otelcol_(receiver_accepted|exporter_sent|processor_dropped)_(spans|metric_points|log_records)"

# ── Instrumentation / App ─────────────────────────────────────
kubectl get instrumentation -A
kubectl describe instrumentation my-instrumentation -n default
kubectl describe pod -l app=my-app | grep -A15 "Init Containers"
kubectl exec -it $(kubectl get pod -l app=my-app -o name | head -1) -- env | grep OTEL
```

---

## 증상별 빠른 참조

| 증상 | 가능한 원인 | 확인 명령어 |
|------|------------|------------|
| 트레이스 없음 | init container 미주입 | `kubectl describe pod -l app=my-app` |
| 트레이스 없음 | OTLP 엔드포인트 오류 | `kubectl exec ... env \| grep OTEL` |
| 트레이스 없음 | Tempo 연결 실패 | Collector 로그 `grep tempo` |
| 트레이스 드랍 | memory_limiter 트리거 | `:8888/metrics` dropped_spans 확인 |
| 메트릭 없음 | Prometheus scrape 미설정 | Prometheus `/targets` 확인 |
| 메트릭 없음 | 이름 변환 오해 | `:8889/metrics` 직접 조회 |
| 로그 없음 | DaemonSet 미배포 | `kubectl get pods -o wide \| grep daemonset` |
| 로그 없음 | Loki 연결 실패 | Collector 로그 `grep loki` |
| 로그-트레이스 미연동 | trace_id가 로그에 없음 | `kubectl logs -l app=my-app \| jq '.trace_id'` |
| Operator CrashLoop | cert-manager 미설치 | `kubectl get pods -n cert-manager` |
| Auto Instrumentation 미작동 | 어노테이션 오류 | `kubectl get pod -o yaml \| grep annotation` |
