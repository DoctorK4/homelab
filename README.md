# 오래된 노트북으로 홈 서버 만들기

> Lenovo IdeaPad 300에 Ubuntu Server를 설치하고, 네트워크·컨테이너·모니터링·장애 대응을 직접 구축하는 기록.

**마지막 문서 업데이트:** 2026-10-28
**프로젝트 상태:** 운영 환경 구축 중

## 문서 안내

- [Ubuntu 설치·SSH·덮개 및 배터리 설정](docs/01-ubuntu-and-ssh.md)
- [Tailscale·Docker·Nginx·Glances](docs/02-tailscale-and-docker.md)
- [Discord 장애·복구 모니터링](docs/03-discord-monitoring.md)
- [문제 해결 기록](troubleshooting/README.md)
- [변경 이력](CHANGELOG.md) · [문서 업데이트 방법](CONTRIBUTING.md)

## 작업 상태와 기록 기준

아래 구축 결과는 제공된 기존 구축기 원문에 근거한다. 이 저장소를 정리하는 과정에서 실제 Lenovo 서버에 접속하거나 구축 결과를 다시 검증하지는 않았다. 현재 저장소에는 문서와 설정 예시가 있으며, 실제 모니터링 스크립트와 완전한 서비스 파일은 포함되어 있지 않다.

| 구분 | 작업 |
|---|---|
| 원문상 완료·동작 확인 | Ubuntu 설치, SSH 키 접속, 덮개 닫힘 후 접속 유지, Tailscale 접속, Docker 기본 실행, Nginx 화면, Glances 웹 UI, Docker 장애·복구 Discord 알림 |
| 구현 완료·실제 조건 미검증 | 배터리 20% 이하 안전 종료, 배터리 30% 이하 및 디스크 90% 이상 경고 |
| 미구현·예정 | 온도 경고, AC 분리 즉시 알림, 외부 Heartbeat, 외부 자동 백업, HTTPS 공개, PostgreSQL·웹 앱, CI/CD |
| 추가 확인 필요 | NordVPN과 Tailscale 간 접속 문제의 근본 원인, 알림 재시도 정책 개선 |

체크박스는 실제 작업과 검증을 마친 뒤 갱신한다. 검증한 날짜·조건·관찰 결과를 해당 문서와 변경 이력에 함께 남긴다.

## 시작한 이유

사용하지 않던 Lenovo 노트북을 24시간 동작하는 홈 서버로 활용하기로 했다. 프론트엔드 개발을 넘어 운영체제, 네트워크, 컨테이너, 배포, 모니터링까지 실제 운영 환경에서 경험하는 것이 목표다.

| 구성 | 사양 |
|---|---|
| 장비 | Lenovo IdeaPad 300 |
| CPU | Intel Pentium N3700 |
| RAM | 4GB |
| SSD | 128GB |
| OS | Ubuntu Server 26.04.1 LTS |
| 연결 | Wi-Fi, SSH, Tailscale |

## 구축 일지

### 1. Ubuntu Server 설치

Mac에서 설치 USB를 준비했다. balenaEtcher에서 문제가 발생해 `dd` 명령으로 USB를 만들었다. 구형 장비의 Legacy BIOS 환경에서 Ubuntu Server를 설치했고, 설치 과정의 Wi-Fi DHCP 지연을 해결하기 위해 설치 후 Netplan으로 무선 인터페이스를 구성했다.

### 2. SSH 원격 관리

Mac에서 Ed25519 SSH 키를 생성하고 서버에 공개키를 배포했다. 공유기 DHCP 예약으로 내부 주소를 고정해 관리하고, SSH 설정에 별칭을 등록해 `ssh lenovo`로 접속할 수 있게 했다. 비밀번호 기반 반복 로그인 대신 키 인증을 활용했다.

### 3. 덮개를 닫아도 동작하도록 설정

노트북 덮개를 닫을 때 발생할 수 있는 절전을 막기 위해 systemd-logind의 덮개 처리 정책을 `ignore`로 변경했다. 덮개를 닫은 상태에서도 원격 접속이 유지되는 것을 확인했다.

### 4. 배터리 보호

노트북 배터리를 일종의 간이 UPS로 활용한다. `/sys/class/power_supply/`에서 AC 연결 여부와 배터리 잔량을 읽는 Bash 스크립트를 만들고 systemd timer로 5분마다 실행하도록 했다. **AC가 분리된 상태에서 배터리 잔량이 20% 이하일 때 안전 종료**하도록 설정했다. 단, 실제 저전력 종료 시험은 아직 진행하지 않았다.

### 5. Tailscale 원격 접속

공유기 SSH 포트를 외부에 직접 공개하는 대신 Tailscale을 도입했다. Mac과 서버를 같은 Tailnet에 연결하고 SSH 별칭 `lenovo-remote`를 설정했다. NordVPN을 켠 상태에서는 SSH 타임아웃이 발생했고, VPN을 끄면 정상 접속되는 것을 확인했다.

### 6. Docker와 Nginx

Docker Engine과 Compose를 설치한 뒤 `nginx:alpine` 컨테이너를 실행했다. 호스트의 8080 포트를 컨테이너의 80 포트에 매핑해 로컬 브라우저에서 기본 Nginx 페이지를 확인했다. 재시작 정책은 `unless-stopped`로 설정했다.

### 7. Glances 모니터링

Glances를 Docker 컨테이너로 실행해 CPU·메모리·디스크·배터리 상태를 웹에서 확인할 수 있게 했다. 웹 포트는 Tailscale 인터페이스 주소에 바인딩해 외부 인터넷에 직접 노출하지 않았다. 관찰 당시 CPU 사용률은 약 8.6%, 메모리는 약 704MB 수준이었다. 측정 시점에 따라 수치는 달라질 수 있다.

### 8. Discord 장애·복구 알림

Bash 스크립트와 systemd timer를 이용해 5분 간격으로 다음 항목을 확인하도록 구성했다.

| 항목 | 경고 조건 | 상태 |
|---|---|---|
| 배터리 | AC 분리 + 잔량 30% 이하 | 구현, 실경고 미검증 |
| 디스크 | 루트 파일시스템 사용률 90% 이상 | 구현, 실경고 미검증 |
| Docker | `glances`, `nginx-test` 실행 중단 | 장애·복구 메시지 실검증 완료 |

이전 상태를 파일에 기록해 동일 장애의 반복 알림을 줄였다. Nginx 컨테이너를 실제로 중단하고 복구하며 Discord에 경고와 복구 메시지가 모두 도착하는 것을 확인했다. Discord 웹후크는 root 전용 설정 파일로 분리했다.

## 현재 구성

```text
MacBook ── SSH / Tailscale ── Lenovo Ubuntu Server
                                  ├─ Docker
                                  │   ├─ Nginx
                                  │   └─ Glances
                                  └─ systemd timers
                                      ├─ 배터리 저전력 종료
                                      └─ 상태 감시 → Discord
```

## 앞으로 할 일

- [ ] CPU 온도 센서를 확인하고 80°C 이상 경고 구현
- [ ] AC 전원 분리 즉시 알림 구현
- [ ] 서버 전체 다운 감지용 외부 Heartbeat 모니터링
- [ ] 설정 파일과 서비스 데이터의 외부 자동 백업
- [ ] HTTPS 서비스 공개 방식 검토 (예: Cloudflare Tunnel)
- [ ] PostgreSQL과 자체 웹 애플리케이션 운영
- [ ] GitHub 기반 CI/CD 자동 배포

## 배운 점

서버는 애플리케이션을 실행하는 것으로 완성되지 않는다. **원격에서 접속할 수 있어야 하고, 상태를 관찰할 수 있어야 하며, 장애가 발생했을 때 알아차리고 복구할 수 있어야 한다.** 제한된 하드웨어로 시작했지만 실제 운영의 기본기를 익히기에는 충분했다.

> 이 저장소에는 공개 가능한 설정 예시와 구축 과정만 기록한다. 웹후크 URL, SSH 개인키, 인증 토큰, 실제 내부 IP 등 민감한 정보는 커밋하지 않는다.
