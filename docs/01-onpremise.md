# 온프레미스 3티어 구축 

## 목적 
온프레미스 환경에서 3티어 아키텍처를 구축해 본다.
  
## 1. WEB 서버

### 서버 정보

- Hostname : **web01**
- IP       : **192.168.111.141**
- OS       : **Ubuntu 24.04**

### Nginx 설치 전 확인

- **hostname** : 웹 서버를 식별하기 위해 `web01`로 호스트 이름 변경
- **ip addr** : 서버에 설정된 IP 주소 확인
- **ip route** : 서버의 게이트웨이 및 라우팅 상태 확인
- **timedatectl** : 서버 로그 시간의 일관성 유지 확인
- **free -h** : 메모리 사용량 확인
- **df -h** : 디스크 사용량 확인

### Nginx 설치

1. **Apache 대신 Nginx를 선택한 이유** :
   * Apache는 클라이언트 요청마다 독립된 프로세스나 스레드를 할당하는 방식을 사용합니다. 따라서 접속자가 많아지면 메모리와 CPU 사용량이 급격히 증가하며, 대규모 동시 접속 환경에서 병목 현상이 발생하기 쉽습니다.
   * Nginx는 적은 수의 프로세스만 유지한 채 비동기 이벤트 기반으로 수많은 요청을 효율적으로 처리하므로, 대량의 동시 접속이 발생해도 일정한 메모리를 유지하며 안정적으로 작동합니다.
  
2. **패키지 목록 업데이트** : `sudo apt update`
3. **Nginx 설치** : `sudo apt install nginx -y`
4. **서비스 상태 확인** : `sudo systemctl status nginx`
5. **프로세스 확인** : `ps -ef | grep nginx` (**root**는 Master 프로세스로 포트와 프로세스를 관리하며, **www-data**는 서비스 전용 계정으로 HTTP 요청을 처리합니다.)
6. **80번 포트 확인** : `sudo ss -tnlp | grep :80`
7. **서버 내부 접속 테스트** : `curl http://localhost`
8. **Nginx 로그 확인** : `ls -lh /var/log/nginx/`
   * **access.log** : Nginx 웹 서버 접속 로그 확인 (`sudo tail -f /var/log/nginx/access.log`)
   * **error.log** : Nginx 웹 서버 에러 로그 확인 (`sudo tail -f /var/log/nginx/error.log`)
9. **설정 검사 명령** : `sudo nginx -t`

### Logrotate 

1. **로그 보관 및 용량 관리 시스템**
2. **Nginx Logrotate 설정 확인** : `cat /etc/logrotate.d/nginx`
3. **날짜별 표시 설정** : `/etc/logrotate.d/nginx` 파일에 `dateext` 및 `dateformat -%Y%m%d` 옵션 추가

### Nginx 버전 노출 확인 및 숨기기

1. **버전 노출 확인** : `curl -I http://localhost`
2. **수정 방법** : `sudo nano /etc/nginx/nginx.conf` 파일을 열어 `http { }` 블록 안에 `server_tokens off;` 추가 후 문법 검사(`sudo nginx -t`) 및 서비스 재적용(`sudo systemctl reload nginx`)

### 장애 테스트

1. **장애 상황** : Nginx 서비스 중지
2. **증상** : 웹 페이지 접속 불가
3. **확인 순서** : 
   * `systemctl status nginx`
   * `ps -ef | grep nginx`
   * `sudo ss -tnlp | grep :80`
   * 웹 브라우저 또는 curl 접속 테스트
4. **원인** : Nginx 서비스 중지
5. **조치** : `sudo systemctl start nginx`
6. **검증** : 서비스 상태 확인, 80번 포트 LISTEN 확인, curl 정상 응답 확인, 웹 접속 확인
7. **배운 점** : WEB 장애 발생 시 **[서비스 ➔ 프로세스 ➔ 포트 ➔ HTTP ➔ 로그]** 순서로 체계적으로 확인해야 합니다.

### 트러블슈팅

  **[이슈] `sudo nginx -t` 입력 시 응답이 지나치게 느린 현상**  
   * **확인 방법** : `time sudo nginx -t` 명령어로 측정 시 약 15초 뒤에 결과 출력
   * **원인** : 초기 서버 설정 시 호스트 이름(hostname)을 변경했으나, `/etc/hosts` 파일 내부의 이름이 동기화되지 않아 이름 조회(DNS/Local resolution) 과정에서 지연 발생
   * **해결 방법** : 서버의 호스트 이름을 변경했다면 `/etc/hosts` 파일도 함께 확인하여 일치하도록 수정해야 합니다.
