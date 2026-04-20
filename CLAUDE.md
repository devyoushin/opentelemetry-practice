# opentelemetry-practice — 프로젝트 가이드

## 프로젝트 설정
- 환경: EKS
- OpenTelemetry Collector 버전: 0.100.x
- OTel Operator 버전: 0.100.x
- 네임스페이스: monitoring
- 앱 이름 컨벤션: my-app
- 백엔드: Grafana Tempo (트레이스), Prometheus (메트릭), Loki (로그)

---

## 디렉토리 구조

```
opentelemetry-practice/
├── CLAUDE.md                  # 이 파일 (자동 로드)
├── .claude/
│   ├── settings.json
│   └── commands/              # /new-doc, /new-runbook, /review-doc, /add-troubleshooting, /search-kb
├── agents/                    # doc-writer, collector-designer, instrumentation-advisor, troubleshooter
├── templates/                 # service-doc, runbook, incident-report
├── rules/                     # doc-writing, opentelemetry-conventions, security-checklist, monitoring
├── collector/                 # Collector 설정 파일
├── app/                       # 계측 예제 애플리케이션
└── *-guide.md                 # 주제별 가이드 문서
```

---

## 커스텀 슬래시 명령어

| 명령어 | 설명 | 사용 예시 |
|--------|------|---------|
| `/new-doc` | 새 가이드 문서 생성 | `/new-doc tail-sampling-processor` |
| `/new-runbook` | 새 런북 생성 | `/new-runbook Collector 메모리 OOM 대응` |
| `/review-doc` | 문서 검토 | `/review-doc collector-guide.md` |
| `/add-troubleshooting` | 트러블슈팅 케이스 추가 | `/add-troubleshooting Span 누락` |
| `/search-kb` | 지식베이스 검색 | `/search-kb OTel Sampling 전략` |

---

## 가이드 문서 목록

| 문서 | 주제 |
|------|------|
| `install.md` | OTel Collector 설치 (Helm + Operator) |
| `architecture-guide.md` | OpenTelemetry 아키텍처 |
| `collector-guide.md` | Collector 파이프라인 설정 |
| `instrumentation-guide.md` | SDK 계측 (자동/수동) |
| `tracing-guide.md` | Distributed Tracing |
| `metrics-guide.md` | OTLP 메트릭 수집 |
| `logging-guide.md` | 로그 수집 (Filelog Receiver) |
| `sampling-guide.md` | Sampling 전략 |
| `backend-integration-guide.md` | 백엔드 연동 (Tempo, Prometheus, Loki) |
| `troubleshooting-guide.md` | 트러블슈팅 |
| `otel-demo-test.md` | OTel Demo 실습 |

---

## 핵심 명령어

```bash
# Collector 상태 확인
kubectl get pods -n monitoring -l app.kubernetes.io/name=opentelemetry-collector

# Collector 내부 메트릭 확인
curl http://collector:8888/metrics | grep otelcol_

# OTLP 테스트 전송 (grpcurl)
grpcurl -plaintext -d '{}' collector:4317 opentelemetry.proto.collector.trace.v1.TraceService/Export

# Collector 설정 검증
otelcontribcol validate --config=collector-config.yaml
```
