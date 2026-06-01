# OpenTelemetry Docs

OpenTelemetry를 처음 보는 사람이 설치부터 계측과 백엔드 연동까지 따라갈 수 있도록 문서를 묶어 둔 디렉터리다.

## 어디서 시작할까

| 순서 | 문서 | 용도 |
|------|------|------|
| 1 | `install.md` | Helm, systemd, Docker Compose 설치 방식 |
| 2 | `architecture-guide.md` | 전체 수집 아키텍처 |
| 3 | `collector-guide.md` | Receiver, Processor, Exporter 파이프라인 |
| 4 | `tracing-guide.md`, `metrics-guide.md`, `logging-guide.md` | Trace/Metric/Log 이해 |
| 5 | `instrumentation-guide.md`, `sampling-guide.md` | 계측과 샘플링 |
| 6 | `backend-integration-guide.md` | Tempo, Prometheus, Loki, Grafana 연동 |
| 7 | `otel-demo-test.md`, `troubleshooting-guide.md` | 실습과 문제 해결 |
| 8 | `rules/README.md` | 문서와 운영 규칙 |
| 9 | `agents/README.md` | AI 작업 지침 |
| 10 | `templates/README.md` | 문서 템플릿 |
| 11 | `../ops/README.md` | 실제 실행 자산과 운영 방법 |

## 문서 구조

| 구분 | 문서 |
|------|------|
| 설치/기초 | `install.md`, `architecture-guide.md`, `collector-guide.md` |
| Signal | `tracing-guide.md`, `metrics-guide.md`, `logging-guide.md` |
| 계측/샘플링 | `instrumentation-guide.md`, `sampling-guide.md` |
| 백엔드/운영 | `backend-integration-guide.md`, `otel-demo-test.md`, `troubleshooting-guide.md` |
| 보조 자료 | `rules/`, `agents/`, `templates/` |

## 읽는 순서

1. `install.md`
2. `architecture-guide.md`
3. `collector-guide.md`
4. `tracing-guide.md`
5. `metrics-guide.md`
6. `logging-guide.md`
7. `instrumentation-guide.md`
8. `sampling-guide.md`
9. `backend-integration-guide.md`
10. `rules/README.md`
11. `agents/README.md`
12. `templates/README.md`

## 관련 경로

- `rules/`는 문서/운영 규칙
- `agents/`는 Claude 작업 지침
- `templates/`는 반복 문서 골격
- `../ops/`는 Collector, Instrumentation, 샘플 앱 매니페스트
