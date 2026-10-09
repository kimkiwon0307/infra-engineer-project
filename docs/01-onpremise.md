
## 1. WEB 서버 기본 설정

### 목적

WEB 서버에 Nginx를 설치하기 전에
서버의 네트워크, 시간, 자원 상태를 먼저 확인한다.

운영 환경에서는 애플리케이션 설치 전에
서버 자체가 정상 상태인지 확인할 필요가 있다.

### 서버 정보

Hostname : web01
IP       : 192.168.111.141
OS       : Ubuntu 24.04

### 확인 항목

hostname
ip addr
ip route
timedatectl
uptime
free -h
df -h

### 각 명령어를 확인한 이유

hostname
- 현재 접속한 서버를 식별하기 위해 확인

ip addr
- 서버에 설정된 IP를 확인

ip route
- 서버의 Gateway 및 Routing 상태 확인

timedatectl
- WEB/WAS/DB 로그 시간의 일관성을 유지하기 위해 확인

free -h
- 메모리 상태 확인

df -h
- 디스크 사용량 확인
