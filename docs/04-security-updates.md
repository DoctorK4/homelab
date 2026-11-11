# Ubuntu 보안 업데이트 점검

## 2026-11-12 실제 서버 점검 (KST)

현재 상태: **백업·자동 업데이트 정책 검증·보안 패치 적용·업데이트 Discord 알림 추가 완료. 수동 재부팅과 재부팅 후 복구 검증은 미수행.**

### 2026-11-12 보안 패치 적용 및 업데이트 알림 추가

- 실제 작업 시간: 04:54:13–05:03:41 KST, 약 9분 28초.
- 관리자 인증 후 설정 백업·압축 검사·SHA-256 검사를 성공한 사용자 출력을 확인했다. 백업은 서버 root 전용이며 시스템 전체 이미지 또는 패키지 자동 복원 수단은 아니다.
- 별도 home-server-security-update.service가 보안 출처만 허용하는 일회성 정책으로 패치를 적용했다. 후보 11개와 필요한 커널 의존 패키지가 설치되었고 linux-image-7.0.0-43-generic 7.0.0-43.43 및 Perl 5.40.1-7ubuntu0.4 설치를 조회했다.
- 종료 후 SSH에서 상태 success, 서비스 Result=success/종료 코드 0, dpkg --audit 이상 없음, 최종 모의 실행의 적용 가능 후보 없음 결과를 확인했다.
- 일회성 정책은 Docker·containerd·systemd·udev·Tailscale·SSH·네트워크 관련 패키지를 제외하고 불필요 패키지 삭제와 needrestart의 서비스 재시작을 억제했다. 기존 일일 업데이트 정책에 영구 제외를 추가하지 않았다.
- 보호 대상 패키지 버전, 컨테이너 ID·시작 시각이 유지되었다는 검증 결과와 기존 Docker·배터리 보호·Discord 감시 타이머의 활성 상태를 확인했다.
- 기존 root 전용 웹후크 설정을 읽는 별도 home-server-update-notify.service/timer를 추가했다. 기존 감시 스크립트·웹후크 설정의 해시 일치를 확인한 사용자 출력을 받았다. 비밀 웹후크를 출력하거나 저장소에 기록하지 않았다.
- 알림은 5분 주기로 재부팅 필요 상태, APT 서비스·수동 업데이트 서비스 실패 및 최신 unattended-upgrades 로그의 오류를 감시한다. 성공한 전달 상태만 저장하여 중복을 줄이고 전송 실패 시 다음 주기에 재시도한다.
- 재부팅 필요 알림의 전송 처리를 포함한 감시 서비스 성공을 확인했다. Discord 앱에서 실제 메시지 수신을 사용자가 확인했는지는 미확인이다. 실제 업데이트 실패를 의도적으로 만들지는 않았으며 실패·복구 및 재시도 동작은 모의 검사 7개로 검증했다.
- 자동 재부팅 false 유지. 실행 커널은 7.0.0-38-generic이며, 새 커널 효력 반영을 위한 수동 재부팅과 재부팅 후 서비스 복구 검증은 미실행이다.

아래는 보안 패치 적용 이전 단계의 기록이다.

### 같은 날 후속 적용 및 검증

사용자가 직접 관리자 인증 후 준비된 도우미를 실행했고 다음 성공 출력을 공유했다.

- 변경 전 /etc/apt 백업 생성 및 SHA-256 일치 확인.
- APT 설정 검증 및 기존 서비스 활성 상태 비교 성공.
- apt-get update --error-on=any 및 unattended-upgrade --dry-run --debug 성공.
- 패키지 설치·서비스 재시작·Discord 설정 변경은 수행하지 않음.

후속 SSH 접속으로 다음 사항을 직접 재확인했다.

- APT::Periodic::Update-Package-Lists=1, Unattended-Upgrade=1.
- Unattended-Upgrade::Automatic-Reboot=false가 유효 설정에 명시됨.
- /var/backups/home-server-auto-updates-* 백업 디렉터리가 root 소유 0700으로 존재.
- Docker, 배터리 보호·Discord 감시·APT 타이머 활성. 기존 두 컨테이너의 가동 시간 유지.
- 목록 갱신 후에도 업데이트 후보 43개, 보안 출처 표시 후보 11개.

백업 압축·해시 검사와 모의 실행 성공은 사용자 터미널 출력으로 확인했다. 백업 내부·모의 실행 상세 계획은 관리자 인증 제한으로 직접 재조회하지 못했다. 기존 보안 업데이트 허용 출처는 유지했다. 수동 패치 설치, 재부팅 및 신규 Discord 알림 연계는 수행하지 않았다. 백업은 APT 설정 범위이며 Docker 데이터나 시스템 전체 백업이 아니다.

아래는 후속 적용 이전의 읽기 전용 점검 기록이다.

LAN SSH 별칭은 시간 초과였으나 Tailscale SSH 별칭으로 공개키 인증에 성공했다. 관리자 권한 명령은 대화형 인증이 필요해 실행하지 못했다. 실제 주소·호스트 식별 정보·웹후크·키는 기록하지 않는다.

| 항목 | 관찰 결과 |
|---|---|
| Ubuntu | 26.04.1 LTS (resolute) |
| 실행 중 커널 | 7.0.0-38-generic |
| apt / unattended-upgrades | 3.2.1ubuntu0.1 / 2.12ubuntu9 설치 |
| 일일 자동 업데이트 | Update-Package-Lists=1, Unattended-Upgrade=1 |
| 허용 출처 | Ubuntu 기본 release 및 security, ESM apps-security/infra-security. 일반 updates 및 제3자 저장소는 허용 목록에 없음 |
| 자동 재부팅 | 명시적 설정 없음. 설치된 unattended-upgrade 코드가 사용하는 기본값 False 확인. 명시적 false 설정 추가는 미실행 |
| 예약 작업 | apt-daily.timer, apt-daily-upgrade.timer 활성화 |
| 최근 실행 결과 | apt-daily / apt-daily-upgrade 모두 Result=success, 종료 코드 0 |
| 자동 업데이트 실적 | 11월 11일 python3-pyasn1 업데이트 후 All upgrades installed 기록 확인 |
| 재부팅 필요 | /run/reboot-required 존재. 사유 목록: linux-image-7.0.0-42-generic, linux-base |
| 패키지 상태 | dpkg --audit 출력 없음 |
| 업데이트 후보 | 기존 캐시 기준 43개, 이 중 security pocket 표시 후보 11개 |
| 기존 서비스 | Docker 활성, nginx-test·glances 실행 중. 배터리 보호·Discord 감시 타이머 활성, 최근 서비스 실행 성공 |
| 실패한 systemd unit | 조회 시점 0개 |

### 업데이트 후보의 한계

관리자 인증 제한으로 apt update를 실행하지 않았다. 저장소 메타데이터 파일의 최근 시각은 11월 11일이며, 후보 개수는 그 캐시 기준이다. 보안 출처 표시가 있는 후보는 bpftool, libperl5.40, linux-generic, linux-headers-generic, linux-image-generic, linux-libc-dev, linux-perf, linux-tools-common, perl-base, perl-modules-5.40, perl이다. 커널 메타패키지 후보 버전은 7.0.0-43.43이다.

이 목록은 실제 자동 업데이트 적용 계획과 같다고 단정하지 않는다. 새 커널 의존 패키지, 보류 정책 및 최신 메타데이터를 반영한 관리자 권한 dry-run이 필요하다. containerd·systemd·Tailscale 등의 일반 업데이트 후보도 있으나 이번 작업에서 적용하지 않았다.

### 수행한 검증

- /etc/os-release, uname, dpkg-query, apt-config dump, apt list --upgradable, dpkg --audit 조회.
- systemctl list-timers / show / is-active / --failed, docker ps 조회.
- 재부팅 필요 파일, 읽기 가능한 APT·unattended-upgrades 기록, 자동 재부팅 기본값 코드 확인.
- 기존 컨테이너를 중단하거나 배터리 임계 조건을 시험하지 않았다. 이번 점검에서 Discord 메시지 수신을 새로 시험하지 않았다.

### 초기 점검 시 미수행 작업과 재개 조건

- 관리자 권한으로 변경 전 설정 백업 및 무결성 검증, 자동 재부팅 false 명시, 최신 목록 갱신·dry-run·필요한 보안 패치 적용.
- 기존 root 전용 Discord 설정을 노출하지 않고 업데이트 실패·재부팅 필요 알림을 연계하고 검증.
- 재부팅은 실행하지 않았다. 커널 업데이트 효력 반영은 별도의 수동 재부팅 및 서비스 복구 확인이 필요하다.

관리자 인증이 필요한 상태이며 비밀번호는 대화에 입력하지 않는다. 설정 도우미를 서버 사용자 홈에 준비하고 Bash 문법 검사만 수행했다. 도우미 실행·백업 생성·설정 적용·dry-run 성공으로 기록하지 않는다. 기존 Docker, 배터리 보호 및 Discord 설정은 변경하지 않았다.

자동 업데이트 동작 기준: [Ubuntu 공식 문서](https://documentation.ubuntu.com/server/how-to/software/automatic-updates/).
