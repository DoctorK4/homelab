# Ubuntu Server 설치와 SSH 원격 접속

## 목표

구형 노트북을 화면 없이 관리할 수 있는 Ubuntu 서버로 전환한다.

## 설치 과정

1. Ubuntu Server 26.04.1 LTS 설치 USB를 Mac에서 준비했다. balenaEtcher 문제가 있어 `dd`를 사용했다.
2. Legacy BIOS로 부팅하고 Ubuntu Server를 설치했다.
3. Wi-Fi 연결이 지연되어 설치 후 Netplan으로 무선 인터페이스를 구성했다.
4. 공유기 DHCP 예약으로 서버의 로컬 주소가 유지되도록 했다.
5. Mac에서 SSH 키를 생성하고 서버에 등록했다.

```bash
ssh-keygen -t ed25519
ssh-copy-id USER@SERVER_LAN_IP
ssh USER@SERVER_LAN_IP
```

`USER`, `SERVER_LAN_IP`는 실제 값으로 대체한다.

## 덮개를 닫아도 서버 유지

`/etc/systemd/logind.conf.d/10-lid.conf`에 다음과 같은 정책을 적용했다.

```ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

## 배터리 보호

배터리와 AC 전원 상태를 `/sys/class/power_supply/`에서 읽어, 방전 중 잔량이 20% 이하가 되면 안전 종료하는 systemd 타이머를 구성했다. 실제 저전력 종료 테스트는 미완료다.
