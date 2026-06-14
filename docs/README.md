# OpenTelemetry Docs

OpenTelemetry를 처음 보는 사람이 설치, 아키텍처, Collector 파이프라인, 계측, 백엔드 연동, 운영 진단까지 순서대로 따라갈 수 있도록 정리한 문서 디렉터리입니다.

## 빠른 길잡이

| 지금 하고 싶은 일 | 열 문서 |
|------|------|
| 전체 흐름을 빠르게 훑기 | [architecture/architecture-guide.md](architecture/architecture-guide.md) |
| 설치 방식을 고르기 | [install/install.md](install/install.md) |
| EKS에 OTel Operator/Collector 설치하기 | [install/install-helm.md](install/install-helm.md) |
| Collector receiver/processor/exporter 이해하기 | [collector/collector-guide.md](collector/collector-guide.md) |
| trace, metric, log 개념 보기 | [signals/tracing-guide.md](signals/tracing-guide.md), [signals/metrics-guide.md](signals/metrics-guide.md), [signals/logging-guide.md](signals/logging-guide.md) |
| 앱에 OpenTelemetry 붙이기 | [instrumentation/instrumentation-guide.md](instrumentation/instrumentation-guide.md) |
| 샘플링 정책 잡기 | [instrumentation/sampling-guide.md](instrumentation/sampling-guide.md) |
| Tempo/Prometheus/Loki/Grafana 연결하기 | [instrumentation/backend-integration-guide.md](instrumentation/backend-integration-guide.md) |
| 끝까지 배포해서 확인하기 | [practice/otel-demo-test.md](practice/otel-demo-test.md) |
| 문제가 생겼을 때 진단하기 | [operations/troubleshooting-guide.md](operations/troubleshooting-guide.md) |

## 추천 읽기 순서

| 순서 | 문서 | 핵심 내용 |
|------|------|------|
| 1 | [install/install.md](install/install.md) | 설치 방식 선택 |
| 2 | [install/install-helm.md](install/install-helm.md) | EKS 기준 Helm 설치 |
| 3 | [architecture/architecture-guide.md](architecture/architecture-guide.md) | 앱, SDK, Collector, backend 관계 |
| 4 | [collector/collector-guide.md](collector/collector-guide.md) | Receiver, Processor, Exporter 파이프라인 |
| 5 | [signals/tracing-guide.md](signals/tracing-guide.md) | trace/span, context propagation, Tempo 조회 |
| 6 | [signals/metrics-guide.md](signals/metrics-guide.md) | OTel metric 모델, Prometheus 연동 |
| 7 | [signals/logging-guide.md](signals/logging-guide.md) | filelog receiver, Loki, trace-log correlation |
| 8 | [instrumentation/instrumentation-guide.md](instrumentation/instrumentation-guide.md) | 자동/수동 계측 |
| 9 | [instrumentation/sampling-guide.md](instrumentation/sampling-guide.md) | head/tail sampling, load balancing |
| 10 | [instrumentation/backend-integration-guide.md](instrumentation/backend-integration-guide.md) | Tempo, Prometheus, Loki, Grafana 데이터소스 |
| 11 | [practice/otel-demo-test.md](practice/otel-demo-test.md) | End-to-End 실습 |
| 12 | [operations/troubleshooting-guide.md](operations/troubleshooting-guide.md) | 증상별 진단 |

## 전체 문서 목록

| 구분 | 문서 |
|------|------|
| 설치 | [install/install.md](install/install.md), [install/install-helm.md](install/install-helm.md), [install/install-systemd.md](install/install-systemd.md), [install/install-docker-compose.md](install/install-docker-compose.md), [install/upgrade/README.md](install/upgrade/README.md) |
| 아키텍처 | [architecture/architecture-guide.md](architecture/architecture-guide.md) |
| Collector | [collector/collector-guide.md](collector/collector-guide.md) |
| Signal | [signals/tracing-guide.md](signals/tracing-guide.md), [signals/metrics-guide.md](signals/metrics-guide.md), [signals/logging-guide.md](signals/logging-guide.md) |
| 구현/연동 | [instrumentation/instrumentation-guide.md](instrumentation/instrumentation-guide.md), [instrumentation/sampling-guide.md](instrumentation/sampling-guide.md), [instrumentation/backend-integration-guide.md](instrumentation/backend-integration-guide.md) |
| 실습 | [practice/otel-demo-test.md](practice/otel-demo-test.md) |
| 운영 | [operations/troubleshooting-guide.md](operations/troubleshooting-guide.md), [../ops/README.md](../ops/README.md) |
| 문서 운영 | [rules/README.md](rules/README.md), [templates/README.md](templates/README.md), [agents/README.md](agents/README.md) |

## 폴더 역할

| 폴더 | 역할 |
|------|------|
| [install/](install/install.md) | 설치 방식별 절차와 업그레이드 |
| [architecture/](architecture/architecture-guide.md) | OpenTelemetry 전체 구조와 데이터 흐름 |
| [collector/](collector/collector-guide.md) | Collector 파이프라인과 component 구성 |
| [signals/](signals/tracing-guide.md) | trace, metric, log signal별 수집/조회 |
| [instrumentation/](instrumentation/instrumentation-guide.md) | 앱 계측, 샘플링, 백엔드 연동 |
| [practice/](practice/otel-demo-test.md) | End-to-End 실습과 검증 |
| [operations/](operations/troubleshooting-guide.md) | 장애 진단과 운영 대응 |
| [rules/](rules/README.md) | 문서 작성, OTel 컨벤션, 보안, 모니터링 규칙 |
| [templates/](templates/README.md) | 서비스 문서, 런북, 장애 보고서 템플릿 |
| [agents/](agents/README.md) | AI가 문서를 작성하거나 진단할 때 참고할 역할별 지침 |
| [../ops/](../ops/README.md) | 실제 배포 가능한 Collector, Instrumentation, 샘플 앱 매니페스트 |

## 관리 원칙

- 개념 설명은 `docs/`에 둡니다.
- 실제 적용 가능한 YAML, Collector 설정, 앱 매니페스트는 `ops/`에 둡니다.
- 반복해서 쓰는 문서 골격은 `templates/`에 둡니다.
- 새 문서를 추가할 때는 이 README의 문서 목록과 추천 읽기 순서를 함께 갱신합니다.
