# 분산 트레이싱 가이드

분산 트레이싱은 여러 서비스에 걸친 단일 요청의 전체 흐름을 추적합니다.

---

## 핵심 개념

### Trace와 Span

```
[Trace — 하나의 요청 전체 흐름]
│
├── Span: API Gateway (200ms)
│     ├── Span: UserService.getUser (80ms)
│     │     └── Span: DB.query (60ms)
│     └── Span: OrderService.getOrders (100ms)
│           └── Span: Cache.get (5ms)
```

| 용어 | 설명 |
|------|------|
| **Trace** | 하나의 요청에서 발생한 모든 Span의 집합. `trace_id`로 식별 |
| **Span** | 단일 작업 단위 (함수 호출, DB 쿼리 등). `span_id`로 식별 |
| **Parent Span** | 현재 Span을 호출한 상위 Span |
| **Root Span** | Trace의 시작점 (parent가 없는 최초 Span) |

---

## Span 속성

```
Span {
  trace_id:       "abc123def456..."     # Trace 전체 식별자
  span_id:        "789xyz..."           # 이 Span의 식별자
  parent_span_id: "111aaa..."           # 부모 Span ID (Root는 없음)
  name:           "GET /api/users"      # 작업 이름
  start_time:     1704844800000000000
  end_time:       1704844800080000000
  status:         OK / ERROR
  attributes: {
    http.method:       "GET"
    http.url:          "/api/users"
    http.status_code:  200
    db.system:         "postgresql"
    db.statement:      "SELECT * FROM users WHERE id = ?"
  }
  events: [                             # Span 내 이벤트 기록
    { name: "cache miss", timestamp: ... }
  ]
}
```

---

## Context Propagation (컨텍스트 전파)

서비스 간에 `trace_id`와 `span_id`를 HTTP 헤더로 전달합니다.

```
Service A                         Service B
   │                                  │
   │  HTTP 요청 + W3C TraceContext 헤더│
   │ ────────────────────────────────→│
   │  traceparent: 00-{trace_id}-{span_id}-01
   │                                  │
```

### W3C TraceContext (표준, 권장)

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
              │  │                                │                │
              │  └─ trace_id (32자)               └─ span_id (16자) └─ flags
              └─ version
```

### B3 (Zipkin 호환)

```
X-B3-TraceId: 4bf92f3577b34da6a3ce929d0e0e4736
X-B3-SpanId:  00f067aa0ba902b7
X-B3-Sampled: 1
```

> OTel은 기본적으로 **W3C TraceContext**를 사용합니다. Istio와 함께 사용할 경우 B3도 지원합니다.

---

## Span Status

| Status | 의미 | 설정 시점 |
|--------|------|-----------|
| `UNSET` | 기본값 (성공으로 간주) | 아무 설정 안 한 경우 |
| `OK` | 명시적 성공 | 중요한 성공 시 수동 설정 |
| `ERROR` | 에러 발생 | 예외/에러 발생 시 반드시 설정 |

---

## 트레이스 확인 (Grafana Tempo)

```bash
# Tempo 포트 포워딩
kubectl port-forward svc/tempo -n monitoring 3200:3200

# Grafana에서 확인
kubectl port-forward svc/grafana -n monitoring 3000:80
# Explore → Tempo → TraceID 입력 또는 검색
```

### Grafana Explore에서 트레이스 조회

```
# TraceQL — Tempo의 쿼리 언어
{ span.http.status_code = 500 }                         # 에러 요청
{ duration > 1s }                                        # 1초 초과 요청
{ resource.service.name = "my-app" && status = error }  # 특정 서비스 에러
```

---

## 트레이스-메트릭 상관관계 (RED Metrics)

Tempo는 트레이스에서 RED 메트릭을 자동 계산합니다.

| 메트릭 | 의미 |
|--------|------|
| **R**ate | 초당 요청 수 |
| **E**rrors | 에러율 |
| **D**uration | 응답 시간 (P50/P95/P99) |

```
Grafana → Explore → Tempo → Service Graph 탭
→ 서비스별 RED 메트릭 자동 시각화
```

---

## 트레이스-로그 연동

트레이스에서 관련 로그로 바로 이동합니다.

```
Grafana → Explore → Tempo → Span 클릭
→ "Logs for this span" 버튼 → Loki에서 해당 trace_id로 자동 조회
```

이를 위해 앱 로그에 `trace_id`를 포함해야 합니다.

```python
# Python 예시 — 로그에 trace_id 자동 삽입
from opentelemetry import trace

span = trace.get_current_span()
ctx = span.get_span_context()
logger.info("Request processed", extra={
    "trace_id": format(ctx.trace_id, '032x'),
    "span_id": format(ctx.span_id, '016x'),
})
```

---

## 참고

- [공식문서 - Traces](https://opentelemetry.io/docs/concepts/signals/traces/)
- [공식문서 - Context Propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
- [W3C TraceContext 스펙](https://www.w3.org/TR/trace-context/)
