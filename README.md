# opentelemetry-practice

EKS + OpenTelemetry Operator/Collector 기준으로 계측, Collector 파이프라인, traces/metrics/logs 수집, 샘플링, 백엔드 연동을 정리한 개인 학습 문서입니다.

## 빠른 시작

- 처음 볼 문서: `docs/install.md`
- 전체 흐름: 설치 -> 아키텍처 -> Collector -> traces/metrics/logs -> 계측/샘플링 -> 백엔드 연동
- AI 작업 지침: `CLAUDE.md`

## 구조

```text
opentelemetry-practice/
├── README.md
├── CLAUDE.md
├── docs/
│   ├── README.md
│   ├── agents/
│   ├── rules/
│   ├── templates/
│   └── *.md
└── ops/
    ├── README.md
    └── config/     # Collector, Instrumentation, 샘플 앱 매니페스트
```

## 학습 경로

| 단계 | 문서 |
|------|------|
| 설치 | `docs/install.md` |
| 핵심 개념 | `docs/architecture-guide.md`, `docs/collector-guide.md`, `docs/tracing-guide.md` |
| Signal | `docs/metrics-guide.md`, `docs/logging-guide.md` |
| 계측/샘플링 | `docs/instrumentation-guide.md`, `docs/sampling-guide.md` |
| 백엔드 | `docs/backend-integration-guide.md` |
| 문제 해결 | `docs/troubleshooting-guide.md`, `docs/otel-demo-test.md` |

## 환경

| 항목 | 값 |
|------|-----|
| Platform | EKS |
| Operator | OpenTelemetry Operator |
| Collector | OpenTelemetry Collector Contrib |
| Namespace | `monitoring` |
| Backends | Tempo, Prometheus, Loki, Grafana |
