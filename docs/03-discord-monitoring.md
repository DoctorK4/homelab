# Bash와 systemd timer로 Discord 장애 알림 만들기

## 요구사항

- 5분마다 배터리·디스크·Docker 상태 검사
- 정상 → 장애 시 경고, 장애 → 정상 시 복구 알림
- 장애 지속 시 중복 메시지 방지
- 웹후크 비밀을 스크립트에 직접 저장하지 않기

## 구성

- 실행 스크립트: `/usr/local/bin/home-server-monitor.sh`
- 웹후크 설정: `/etc/home-server-monitor/discord.env` (root 전용, `0600`)
- 상태 저장: `/var/lib/home-server-monitor/`
- 서비스: `home-server-monitor.service`
- 타이머: `home-server-monitor.timer`

```ini
[Unit]
Description=Run Lenovo Home Server Monitor Every 5 Minutes

[Timer]
OnBootSec=1min
OnUnitInactiveSec=5min
AccuracySec=30s
Unit=home-server-monitor.service

[Install]
WantedBy=timers.target
```

## 실제 문제 해결: status=203/EXEC

서비스를 처음 실행할 때 `status=203/EXEC` 오류가 발생했다. `journalctl` 로그에 실행 파일 `/usr/local/bin/home-server-monitor.sh`가 없다고 표시되었다. 실제로 파일이 생성되지 않은 상태였다.

스크립트를 생성하고 소유자·실행 권한을 설정한 뒤 재실행했다.

```bash
sudo chown root:root /usr/local/bin/home-server-monitor.sh
sudo chmod 755 /usr/local/bin/home-server-monitor.sh
sudo bash -n /usr/local/bin/home-server-monitor.sh
sudo systemctl reset-failed home-server-monitor.service
sudo systemctl start home-server-monitor.service
sudo systemctl status home-server-monitor.service --no-pager
```

서비스가 `inactive (dead)`로 표시되더라도 `Type=oneshot` 서비스가 `status=0/SUCCESS`로 종료했다면 정상이다.

## 구현 상태와 검증 범위\n\n| 항목 | 상태 | 검증 |\n|---|---|---|\n| Discord 웹후크 연결 | 완료 | 테스트 메시지 수신 확인 |\n| 5분 주기 systemd 감시 | 완료 | 서비스 `0/SUCCESS`, 타이머 활성화 확인 |\n| Docker 컨테이너 중단·복구 알림 | 완료 | 경고 및 복구 메시지 실제 수신 |\n| 배터리 30% 이하 경고 (AC 분리 시) | 구현 완료 | 실제 임계치 테스트 미실시 |\n| 디스크 90% 이상 경고 | 구현 완료 | 실제 임계치 테스트 미실시 |\n| CPU 온도 80°C 이상 경고 | 확인 필요 | 별도 작업 요청 이력 있으나 최종 적용 여부 미확인 |\n| AC 전원 분리 즉시 알림 | 확인 필요 | 배터리 30% 이하 경고와 별개이며 적용 여부 미확인 |\n\n**주의:** 배터리 경고는 `AC 분리 AND 배터리 잔량 ≤30%` 조건이다. 전원이 분리되는 즉시 보내는 알림과 동일하지 않다. 또한 서버 자체가 종료되면 이 스크립트는 알림을 보낼 수 없으므로 외부 Heartbeat가 별도로 필요하다.\n\n## 장애 및 복구 실험

1. `docker stop nginx-test`로 Nginx를 중단했다.
2. 모니터링 서비스를 실행해 Discord 경고 메시지를 받았다.
3. `docker start nginx-test`로 복구했다.
4. 다시 모니터링 서비스를 실행해 Discord 복구 메시지를 받았다.

**결과:** Docker 중단 및 복구 감지가 정상 작동했다. 배터리와 디스크의 임계치 알림은 아직 실험하지 않았다.

## 남은 개선점

- CPU 온도 80°C 경고
- AC 전원 분리 즉시 알림
- 서버 전체가 종료될 때 외부에서 감지하는 Heartbeat
- Discord API 실패 및 장시간 네트워크 장애에 대한 재시도 정책 개선


## 2026-11-12 업데이트 전용 알림 감시 추가

기존 Docker·배터리·디스크 감시 스크립트는 유지하고, 기존 root 전용 웹후크 설정을 재사용하는 home-server-update-notify.service 및 home-server-update-notify.timer를 추가했다.

- 5분 주기로 재부팅 필요, APT 및 수동 보안 업데이트 서비스 실패, 최신 unattended-upgrades 로그 오류를 확인한다.
- 전달 성공 후 상태를 저장해 동일 알림을 줄인다. 전송 실패 시 다음 주기에 재시도한다. 실패 후 성공 실행이나 재부팅 필요 표시 해제 시 복구를 알린다.
- 알림 서비스 종료 코드 0 및 타이머 활성화를 확인했다. 기존 감시 스크립트와 웹후크 설정은 설치 전후 해시가 같았다(사용자 실행 출력).
- 재부팅 필요 알림 전송 처리는 성공했다. Discord 앱의 실제 메시지 수신은 사용자 확인 전이며, 업데이트 실패를 실제로 발생시키는 시험은 하지 않았다. 실패·복구·재시도 등은 모의 검사로 확인했다.
- 웹후크 URL·HTTP 오류의 상세 문자열·원문 로그는 알림이나 공개 기록에 넣지 않는다.

보안 패치 적용 결과와 미실행 항목은 [보안 업데이트 기록](04-security-updates.md)을 참고한다.


## 2026-11-12 알림 수신 및 재부팅 후 검증

사용자가 재부팅 필요 알림의 실제 수신을 확인했다. 수동 재부팅 후 기존 Discord 감시와 업데이트 알림 서비스의 종료 코드 0, 관련 타이머 활성화를 SSH로 확인했다. 재부팅 필요 표시는 해제되었다. 업데이트 알림 서비스는 부팅 후 정상 실행했으나, Discord 앱에서 복구 알림을 실제로 수신했는지는 사용자 확인 전이다.
