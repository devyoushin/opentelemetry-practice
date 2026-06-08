# 문서와 운영 규칙

OpenTelemetry 문서와 운영 절차를 쓸 때 지켜야 할 기준입니다. 새 문서를 추가하거나 기존 문서를 고칠 때 먼저 확인합니다.

## 문서 목록

| 문서 | 용도 |
|------|------|
| [doc-writing.md](doc-writing.md) | 문서 구조, 코드 예시, 금지 사항 |
| [opentelemetry-conventions.md](opentelemetry-conventions.md) | span 이름, attribute, resource, pipeline 컨벤션 |
| [monitoring.md](monitoring.md) | Collector 자체 모니터링, 알림, SLO, 대시보드 기준 |
| [security-checklist.md](security-checklist.md) | 인증/인가, 민감 정보, 네트워크, SDK 보안 점검 |

## 적용 순서

1. 새 문서를 만들 때 [doc-writing.md](doc-writing.md)를 확인합니다.
2. Collector와 signal 흐름을 설명할 때 [opentelemetry-conventions.md](opentelemetry-conventions.md)를 따릅니다.
3. 운영 알림과 대시보드를 정리할 때 [monitoring.md](monitoring.md)를 기준으로 맞춥니다.
4. 접근 권한, 민감 정보, 외부 전송 설정은 [security-checklist.md](security-checklist.md)로 점검합니다.

## 관련 문서

- [../README.md](../README.md)
- [../agents/README.md](../agents/README.md)
- [../templates/README.md](../templates/README.md)
