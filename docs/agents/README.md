# AI 작업 지침

OpenTelemetry 문서를 AI와 함께 작성하거나 점검할 때 쓰는 역할별 지침입니다. 일반 학습 문서가 아니라, 문서 작성/설계/진단 작업의 기준으로 사용합니다.

## 문서 목록

| 문서 | 용도 |
|------|------|
| [doc-writer.md](doc-writer.md) | 새 가이드, README, 운영 문서를 작성할 때의 기준 |
| [collector-designer.md](collector-designer.md) | Collector receiver/processor/exporter 파이프라인 설계 기준 |
| [instrumentation-advisor.md](instrumentation-advisor.md) | 자동 계측과 수동 계측 전략을 정리하는 기준 |
| [troubleshooter.md](troubleshooter.md) | trace/metric/log 수집 문제를 진단하는 기준 |

## 사용 기준

| 작업 | 먼저 볼 문서 |
|------|------|
| 새 문서 작성 | [doc-writer.md](doc-writer.md) |
| Collector 설정 설계 또는 리뷰 | [collector-designer.md](collector-designer.md) |
| 앱 계측 방식 결정 | [instrumentation-advisor.md](instrumentation-advisor.md) |
| 장애 분석, 재현, 원인 정리 | [troubleshooter.md](troubleshooter.md) |

## 관련 문서

- [../README.md](../README.md)
- [../rules/README.md](../rules/README.md)
- [../templates/README.md](../templates/README.md)
- [../../CLAUDE.md](../../CLAUDE.md)
