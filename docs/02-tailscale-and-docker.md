# Tailscale과 Docker로 원격 홈 서버 만들기

## Tailscale

공유기 포트포워딩 없이 원격 SSH에 접근하기 위해 Tailscale을 사용했다. Mac의 SSH 설정에서 로컬 접속용 `lenovo`, Tailscale 접속용 `lenovo-remote` 별칭을 구분했다.

**문제:** NordVPN이 활성화된 Mac에서 Tailscale SSH 접속이 타임아웃되었다. NordVPN을 비활성화하자 접속이 정상화되었다. 근본적인 라우팅 충돌 원인은 아직 분석하지 않았다.

## Docker 설치 후 Nginx 실행

Docker Engine과 Compose v2를 설치하고 `hello-world` 실행으로 기본 동작을 검증했다.

```yaml
services:
  nginx:
    image: nginx:alpine
    container_name: nginx-test
    ports:
      - "8080:80"
    restart: unless-stopped
```

```bash
docker compose up -d
docker ps
```

LAN의 `http://SERVER_LAN_IP:8080`에서 Nginx 기본 화면을 확인했다.

## Glances

Glances를 컨테이너로 실행하고 웹 UI를 Tailscale 주소에만 바인딩했다. 실제 주소는 공개 저장소에 기록하지 않는다.

```yaml
services:
  glances:
    image: nicolargo/glances:latest-full
    container_name: glances
    restart: unless-stopped
    pid: host
    environment:
      - GLANCES_OPT=-w
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /etc/os-release:/etc/os-release:ro
    ports:
      - "TAILSCALE_IP:61208:61208"
```

> Docker 소켓 마운트는 보안상 민감하므로 신뢰할 수 있는 이미지만 사용하고 외부 공개를 피해야 한다.
