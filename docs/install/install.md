# OpenTelemetry 설치 안내

OpenTelemetry를 실행할 환경에 맞게 설치 문서를 고르는 입구입니다. EKS 실습은 Helm 설치를 기본 경로로 봅니다.

## 어떤 문서를 볼까

| 상황 | 문서 | 설명 |
|------|------|------|
| EKS/Kubernetes에 설치 | [install-helm.md](install-helm.md) | OTel Operator와 Collector를 Helm/YAML로 설치 |
| VM 또는 단일 서버에 설치 | [install-systemd.md](install-systemd.md) | Collector를 systemd 서비스로 관리 |
| 로컬에서 파이프라인만 검증 | [install-docker-compose.md](install-docker-compose.md) | Docker Compose로 Collector와 backend를 빠르게 실행 |
| 버전 변경 또는 롤백 | [upgrade/README.md](upgrade/README.md) | Helm, systemd, Docker Compose 업그레이드/롤백 |

## EKS 기준 진행 순서

1. [install-helm.md](install-helm.md)로 Operator와 Collector를 설치합니다.
2. [../architecture/architecture-guide.md](../architecture/architecture-guide.md)로 전체 수집 구조를 확인합니다.
3. [../collector/collector-guide.md](../collector/collector-guide.md)로 Collector 파이프라인을 이해합니다.
4. [../instrumentation/instrumentation-guide.md](../instrumentation/instrumentation-guide.md)로 앱 자동 계측을 적용합니다.
5. [../instrumentation/backend-integration-guide.md](../instrumentation/backend-integration-guide.md)로 Tempo, Prometheus, Loki, Grafana를 연결합니다.
6. [../practice/otel-demo-test.md](../practice/otel-demo-test.md)로 End-to-End 검증을 수행합니다.

## 설치 후 확인할 것

| 확인 항목 | 관련 문서 |
|------|------|
| Collector Pod와 CRD 상태 | [install-helm.md](install-helm.md) |
| trace가 Tempo에 들어오는지 | [../signals/tracing-guide.md](../signals/tracing-guide.md) |
| metric이 Prometheus에 보이는지 | [../signals/metrics-guide.md](../signals/metrics-guide.md) |
| log가 Loki에 보이는지 | [../signals/logging-guide.md](../signals/logging-guide.md) |
| 문제가 생겼을 때 | [../operations/troubleshooting-guide.md](../operations/troubleshooting-guide.md) |
