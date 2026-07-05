# opentelemetry-practice

EKS 환경에서 OpenTelemetry Operator와 Collector를 사용해 traces, metrics, logs를 수집하고 Grafana Tempo, Prometheus, Loki로 연동하는 과정을 정리한 학습 문서입니다.

## 먼저 볼 문서

| 목적 | 문서 |
|------|------|
| 전체 문서 목차 보기 | [docs/README.md](docs/README.md) |
| 설치 방식 고르기 | [docs/01-installation/install.md](docs/01-installation/install.md) |
| EKS에 Helm으로 설치하기 | [docs/01-installation/install-helm.md](docs/01-installation/install-helm.md) |
| 전체 아키텍처 이해하기 | [docs/02-architecture/architecture-guide.md](docs/02-architecture/architecture-guide.md) |
| Collector 파이프라인 이해하기 | [docs/05-collector/collector-guide.md](docs/05-collector/collector-guide.md) |
| 직접 실습하기 | [docs/06-practice/otel-demo-test.md](docs/06-practice/otel-demo-test.md) |
| 장애를 진단하기 | [docs/07-operations/troubleshooting-guide.md](docs/07-operations/troubleshooting-guide.md) |

## 추천 학습 순서

1. [설치 안내](docs/01-installation/install.md)
2. [아키텍처](docs/02-architecture/architecture-guide.md)
3. [Collector 파이프라인](docs/05-collector/collector-guide.md)
4. [Tracing](docs/03-signals/tracing-guide.md), [Metrics](docs/03-signals/metrics-guide.md), [Logging](docs/03-signals/logging-guide.md)
5. [계측](docs/04-instrumentation/instrumentation-guide.md), [샘플링](docs/04-instrumentation/sampling-guide.md)
6. [백엔드 연동](docs/04-instrumentation/backend-integration-guide.md)
7. [End-to-End 실습](docs/06-practice/otel-demo-test.md)
8. [트러블슈팅](docs/07-operations/troubleshooting-guide.md)

## 디렉터리 구조

```text
opentelemetry-practice/
├── README.md
├── CLAUDE.md       # AI 작업 지침
├── docs/
│   ├── README.md  # 문서 전체 목차
│   ├── install/   # 설치/업그레이드
│   ├── agents/    # AI 역할별 작업 지침
│   ├── rules/     # 문서/운영 규칙
│   ├── templates/ # 서비스 문서, 런북, 장애 보고서 템플릿
│   └── *.md       # 주제별 가이드
└── ops/
    ├── README.md  # 운영 자산 설명
    └── config/    # Collector, Instrumentation, 샘플 앱 매니페스트
```

## 문서 분류

| 분류 | 내용 |
|------|------|
| 설치 | Helm, systemd, Docker Compose, 업그레이드 |
| 핵심 개념 | 아키텍처, Collector, trace/metric/log |
| 구현 | 계측, 샘플링, 백엔드 연동 |
| 실습/운영 | End-to-End 테스트, 트러블슈팅, 운영 자산 |
| 문서 운영 | 작성 규칙, 템플릿, AI 작업 지침 |

## 환경

| 항목 | 값 |
|------|-----|
| Platform | EKS |
| Operator | OpenTelemetry Operator |
| Collector | OpenTelemetry Collector Contrib |
| Namespace | `monitoring` |
| Backends | Tempo, Prometheus, Loki, Grafana |

## 운영 자산

실제 적용 가능한 Collector, Instrumentation, 샘플 앱 매니페스트는 [ops/README.md](ops/README.md)와 `ops/config/` 아래에 둡니다.
