# End-to-End 실습: App → Collector → Tempo / Prometheus / Loki

앱 배포부터 트레이스, 메트릭, 로그 확인까지 전체 흐름을 실습합니다.

---

## 사전 준비 체크리스트

- [ ] EKS 클러스터 접속 확인: `kubectl get nodes`
- [ ] OTel Operator 설치 완료: `kubectl get pods -n monitoring | grep operator`
- [ ] Collector (Gateway + DaemonSet) 배포 완료: `kubectl get pods -n monitoring`
- [ ] 백엔드 서비스 확인: `kubectl get svc -n monitoring | grep -E "tempo|prometheus|loki|grafana"`

---

## Step 1: Collector 배포

```bash
# Gateway Collector (앱 OTLP 수신 → Tempo / Prometheus / Loki)
kubectl apply -f collector/otelcol-deployment.yaml

# DaemonSet Collector (노드 로그 + kubelet 메트릭)
kubectl apply -f collector/otelcol-daemonset.yaml

# 상태 확인
kubectl get pods -n monitoring
kubectl logs -n monitoring deployment/otelcol-deployment --tail=10
```

**예상 결과:**
```
Everything is ready. Begin running and processing data.
```

---

## Step 2: Auto Instrumentation CR 생성

```bash
kubectl apply -f collector/instrumentation.yaml

# 생성 확인
kubectl get instrumentation -n default
# NAME                 AGE
# my-instrumentation   5s
```

---

## Step 3: 샘플 앱 배포

```bash
kubectl apply -f app/deployment.yaml
kubectl apply -f app/service.yaml

# Pod 확인
kubectl get pods -l app=my-app
```

앱에는 이미 Auto Instrumentation 어노테이션이 포함되어 있습니다.
init container 주입 여부를 확인합니다:

```bash
kubectl describe pod -l app=my-app | grep -A10 "Init Containers"
# opentelemetry-auto-instrumentation 컨테이너가 보여야 함

# 환경변수 주입 확인
kubectl exec -it $(kubectl get pod -l app=my-app -o name | head -1) -- env | grep OTEL
# OTEL_EXPORTER_OTLP_ENDPOINT=http://otelcol-deployment.monitoring:4317
# OTEL_SERVICE_NAME=my-app
```

---

## Step 4: 트레이스 생성 (요청 발생)

```bash
# 클러스터 내부에서 앱으로 요청 전송
kubectl run curl-test --image=curlimages/curl -it --rm -- sh -c \
  "for i in \$(seq 1 30); do curl -s http://my-app/api/users; echo; done"
```

---

## Step 5: 트레이스 확인 (Grafana Tempo)

```bash
# Grafana 포트 포워딩
kubectl port-forward svc/grafana -n monitoring 3000:80 &
```

Grafana (`http://localhost:3000`) 에서:
```
Explore → 데이터소스: Tempo
Query type: Search
Service name: my-app
Run query → 트레이스 목록 확인
```

TraceQL로 직접 조회:
```
{ resource.service.name = "my-app" }           # 모든 트레이스
{ duration > 500ms }                            # 느린 요청
{ status = error }                              # 에러 트레이스
{ span.http.target = "/api/users" }             # 특정 엔드포인트
```

**확인 포인트:**
- 트레이스 목록에 `my-app` 서비스가 나타나는지
- Span 상세에서 `k8s.namespace.name`, `k8s.pod.name` 속성이 붙어있는지
- 응답 시간 분포 확인

---

## Step 6: 트레이스 → 로그 이동

Tempo에서 Span을 클릭하면 "Logs for this span" 버튼이 나타납니다.

```
Tempo Span 클릭
  → "Logs for this span" 버튼 클릭
  → Loki Explore로 자동 이동 (trace_id로 필터링된 로그)
```

로그에서 직접 조회:
```
# Loki LogQL
{k8s_namespace_name="default", k8s_container_name="my-app"}
| json
| trace_id != ""
```

**확인 포인트:**
- 로그에 `trace_id`, `span_id` 필드가 포함되어 있는지
- Tempo ↔ Loki 이동이 동작하는지

---

## Step 7: 메트릭 확인 (Grafana + Prometheus)

```bash
# Prometheus 포트 포워딩
kubectl port-forward svc/prometheus-server -n monitoring 9090:80 &
```

Grafana (`http://localhost:3000`) 에서:
```
Explore → 데이터소스: Prometheus
```

유용한 PromQL 쿼리:
```
# OTel SDK가 생성한 HTTP 요청 수
rate(http_server_duration_milliseconds_count{service_name="my-app"}[5m])

# 응답 시간 P95
histogram_quantile(0.95, rate(http_server_duration_milliseconds_bucket{service_name="my-app"}[5m]))

# Pod CPU 사용률 (kubeletstats)
rate(k8s_pod_cpu_time_total{k8s_namespace_name="default"}[5m])

# Collector가 수신한 스팬 수
rate(otelcol_receiver_accepted_spans_total[5m])
```

---

## Step 8: 에러 트레이스 확인

에러를 강제로 발생시킵니다.

```bash
kubectl run curl-test --image=curlimages/curl -it --rm -- sh -c \
  "for i in \$(seq 1 15); do curl -s http://my-app/api/not-found; echo; done"
```

Tempo에서 에러 트레이스 조회:
```
Explore → Tempo
Query: { status = error }
→ 에러가 발생한 트레이스만 필터링됨
→ Span의 status = ERROR, http.status_code = 404 확인
```

---

## Step 9: Collector 수집 현황 확인

```bash
kubectl port-forward -n monitoring deployment/otelcol-deployment 8888:8888 &

# 수신한 스팬 수
curl -s http://localhost:8888/metrics | grep otelcol_receiver_accepted_spans_total

# Tempo로 전송한 스팬 수
curl -s http://localhost:8888/metrics | grep otelcol_exporter_sent_spans_total

# 드랍된 스팬 (0이어야 정상)
curl -s http://localhost:8888/metrics | grep otelcol_processor_dropped_spans

# 메모리 사용량
curl -s http://localhost:8888/metrics | grep otelcol_process_memory_rss
```

---

## 검증 체크리스트

```bash
# 자동화 검증 스크립트
echo "=== OTel E2E 검증 ==="

# 1. Operator 파드 상태
echo -n "[1] OTel Operator Running: "
kubectl get pods -n monitoring -l app.kubernetes.io/name=opentelemetry-operator \
  --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l | tr -d ' '
echo " pods"

# 2. Gateway Collector 상태
echo -n "[2] Gateway Collector Running: "
kubectl get pods -n monitoring -l app.kubernetes.io/name=otelcol-deployment \
  --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l | tr -d ' '
echo " pods"

# 3. DaemonSet Collector 상태
echo -n "[3] DaemonSet Collector (노드당 1개): "
DESIRED=$(kubectl get daemonset -n monitoring otelcol-daemonset -o jsonpath='{.status.desiredNumberScheduled}' 2>/dev/null)
READY=$(kubectl get daemonset -n monitoring otelcol-daemonset -o jsonpath='{.status.numberReady}' 2>/dev/null)
echo "${READY}/${DESIRED}"

# 4. 샘플 앱 상태
echo -n "[4] 샘플 앱 Running: "
kubectl get pods -l app=my-app --field-selector=status.phase=Running \
  --no-headers 2>/dev/null | wc -l | tr -d ' '
echo " pods"

# 5. Auto Instrumentation 주입 여부
echo -n "[5] Init Container 주입: "
kubectl get pod -l app=my-app -o jsonpath='{.items[0].spec.initContainers[*].name}' 2>/dev/null
echo ""

# 6. Collector 수신 스팬 수
echo -n "[6] Collector 수신 스팬 수: "
kubectl port-forward -n monitoring deployment/otelcol-deployment 8888:8888 > /dev/null 2>&1 &
PF_PID=$!
sleep 2
curl -s http://localhost:8888/metrics 2>/dev/null | \
  grep "otelcol_receiver_accepted_spans_total" | \
  awk '{sum += $2} END {print sum " spans"}'
kill $PF_PID 2>/dev/null

# 7. Instrumentation CR 존재
echo -n "[7] Instrumentation CR: "
kubectl get instrumentation my-instrumentation -n default --no-headers 2>/dev/null | \
  awk '{print $1}' || echo "NOT FOUND"

echo ""
echo "=== 검증 완료 ==="
```

---

## 전체 흐름 요약

```
[my-app Pod]
  OTel Python Agent 자동 주입 (init container)
   │ OTLP gRPC → :4317
   ▼
[otelcol-deployment (Gateway)]
  k8sattributes → namespace, pod, deployment 자동 부착
  batch → 배치 전송
   ├── traces  ──▶ Tempo     → Grafana Explore (TraceQL)
   ├── metrics ──▶ Prometheus → Grafana Dashboard (PromQL)
   └── logs    ──▶ Loki       → Grafana Explore (LogQL + trace_id 연결)

[otelcol-daemonset (DaemonSet)]
  filelog → 노드 컨테이너 로그 수집 → Loki
  kubeletstats → 노드/Pod 메트릭 → Gateway → Prometheus
```

---

## 실습 종료 후 정리

```bash
# 샘플 앱 삭제
kubectl delete -f app/deployment.yaml
kubectl delete -f app/service.yaml

# Instrumentation CR 삭제
kubectl delete -f collector/instrumentation.yaml

# Collector 삭제
kubectl delete -f collector/otelcol-deployment.yaml
kubectl delete -f collector/otelcol-daemonset.yaml

# OTel Operator 삭제
helm uninstall opentelemetry-operator -n monitoring

# cert-manager 삭제 (필요한 경우)
helm uninstall cert-manager -n cert-manager
kubectl delete namespace cert-manager

# 포트 포워드 프로세스 종료
pkill -f "kubectl port-forward"
```

---

## 자주 막히는 지점

| 단계 | 증상 | 해결 |
|---|---|---|
| Step 1 | Collector `CrashLoopBackOff` | cert-manager 설치 및 Operator 상태 확인 |
| Step 3 | init container 주입 안 됨 | Instrumentation CR 존재 여부, 어노테이션 키 확인 |
| Step 5 | 트레이스 없음 | OTLP 엔드포인트 환경변수, Tempo 서비스 이름 확인 |
| Step 6 | 로그-트레이스 연동 안 됨 | Loki Derived Fields 설정, 로그에 trace_id 포함 여부 확인 |
| Step 7 | 메트릭 없음 | Prometheus scrape 설정, Collector `:8889` 엔드포인트 확인 |
| 전체 | 스팬 드랍 | Collector `:8888/metrics`에서 `dropped_spans` 확인 |

더 자세한 문제 해결은 [troubleshooting-guide.md](./troubleshooting-guide.md) 참고.
