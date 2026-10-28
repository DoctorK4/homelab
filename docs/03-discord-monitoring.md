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

## 장애 및 복구 실험

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
