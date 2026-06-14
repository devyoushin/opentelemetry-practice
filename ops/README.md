# OpenTelemetry Ops

OpenTelemetry를 실제로 배포하거나 실습할 때 사용하는 설정과 Kubernetes 매니페스트를 두는 공간입니다. 개념 설명은 `docs/`에, 적용 가능한 실행 자산은 `ops/`에 둡니다.

## 폴더 구조

| 폴더 | 내용 |
|------|------|
| `config/collector/` | Collector Deployment/DaemonSet, Instrumentation CR |
| `config/app/` | 계측된 샘플 애플리케이션 매니페스트 |

## 관련 문서

| 작업 | 문서 |
|------|------|
| 설치 방식 선택 | [../docs/install/install.md](../docs/install/install.md) |
| EKS Helm 설치 | [../docs/install/install-helm.md](../docs/install/install-helm.md) |
| Collector 파이프라인 이해 | [../docs/collector/collector-guide.md](../docs/collector/collector-guide.md) |
| 앱 자동 계측 | [../docs/instrumentation/instrumentation-guide.md](../docs/instrumentation/instrumentation-guide.md) |
| End-to-End 실습 | [../docs/practice/otel-demo-test.md](../docs/practice/otel-demo-test.md) |
| 문제 해결 | [../docs/operations/troubleshooting-guide.md](../docs/operations/troubleshooting-guide.md) |

## 관리 원칙

- 재사용 가능한 매니페스트와 설정 예시는 이 디렉터리에 둡니다.
- 문서 본문에는 핵심 스니펫만 넣고, 전체 적용 파일은 `ops/config/`를 기준으로 관리합니다.
- 설정을 바꾸면 관련 문서의 명령어와 파일 경로도 함께 확인합니다.
