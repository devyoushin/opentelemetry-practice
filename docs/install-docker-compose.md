# OpenTelemetry Docker Compose 설치

로컬에서 Collector 파이프라인과 backend 연동을 검증할 때 사용한다.

## 대상

- `otelcol`
- 샘플 앱

## 절차

1. `compose.yaml`에 Collector 이미지와 backend를 정의한다.
2. 설정 파일을 mount 한다.
3. `docker compose up -d`로 올린다.
4. `docker compose logs -f`와 `docker compose ps`를 확인한다.

## 확인 명령

```bash
docker compose ps
curl http://localhost:8888/metrics | head
```

## 운영 포인트

- 로컬 실험은 OTLP gRPC/HTTP 수신 포트와 export target만 확인해도 충분하다.
- 자동 계측 데모는 샘플 앱을 별도 서비스로 둔다.
