# 문서 작성 규칙 — opentelemetry-practice

## 기본 원칙
1. 모든 문서는 한국어로 작성하되, 기술 용어(Span, Trace, Collector, Exporter, Receiver 등)는 영어 그대로 사용
2. 제목은 `# [주제] — OpenTelemetry [문서유형]` 형식으로 통일
3. 문서 상단에 작성일, 작성자, 검토자 메타데이터 포함
4. 코드 블록에는 반드시 언어 지정 (```yaml, ```go, ```python, ```bash)
5. 언어별 계측 코드 예시 포함

## 문서 구조
- **개요**: 목적과 수집 신호 범위 (3-5문장)
- **계측 구성**: SDK 설정 및 Collector 파이프라인
- **신호별 상세**: Traces / Metrics / Logs 각각 설명
- **운영**: 체크리스트 및 명령어
- **트러블슈팅**: 증상-원인-해결 테이블

## Collector 설정 문서화 규칙
- 전체 YAML 예시 포함 (주석 상세 기술)
- 각 receiver/processor/exporter 목적 설명
- 파이프라인 연결 흐름 텍스트 다이어그램 제공

## 계측 코드 문서화 규칙
- 자동 계측과 수동 계측 구분 명시
- Span attribute 목록 테이블로 제공
- Sampling 설정 및 근거 설명

## 금지 사항
- API 키, 토큰 하드코딩 금지
- 프로덕션 엔드포인트 문서에 직접 포함 금지
- 검증되지 않은 계측 코드 예시 금지
