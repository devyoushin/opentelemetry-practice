# OpenTelemetry systemd 설치

단일 VM이나 베어메탈에서 `otelcol-contrib`를 systemd 서비스로 관리할 때 사용한다. OTel Operator는 대상이 아니고 Collector만 관리한다.

## 대상

- `otelcol.service`

## 준비물

- `otelcol-contrib` 바이너리
- 설정 파일: `/etc/otelcol/config.yaml`
- 데이터/상태 디렉터리: `/var/lib/otelcol`

## 절차

1. 바이너리를 설치한다.
2. 설정 파일을 배치한다.
3. unit 파일을 `/etc/systemd/system/otelcol.service`에 둔다.
4. `systemctl enable --now otelcol`로 시작한다.
5. `journalctl -u otelcol -f`로 로그를 본다.

## 확인 명령

```bash
systemctl status otelcol
curl http://localhost:8888/metrics | head
```

## 운영 포인트

- receiver / processor / exporter 경로를 설정 파일에서 분리한다.
- backend endpoint는 환경 변수보다 설정 파일로 관리하는 편이 읽기 쉽다.
- instrumentation 자동 주입은 Kubernetes 전용 기능이므로 systemd 문서 범위 밖이다.
