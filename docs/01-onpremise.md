# 온프레미스 3티어 구축 

## 목적 
온프레미스 환경에서 3티어 아키텍처를 구축해 본다.

## 아키텍처 
![On-Premise 3-Tier Architecture](/images/On-Premise 3-Tier Architecture.drawio.png)
## 1. WEB 서버

### 서버 정보

- Hostname : **web01**
- IP       : **192.168.111.141**
- OS       : **Ubuntu 24.04**

### hostname 설정
 1. hostname web01로 만들기
 2. sudo hostname 확인
 3. sudo hostnamectl set-hostname web01

### timedatectl 설정
  1. timedatectl로 local time 확인
  2. sudo timedatectl set-timezone Asia/Seoul 로 Local time 변경
  3. 장애 로그 시간을 맞추기 위해 해야한다.
  
### Nginx 설치

1. **Apache 대신 Nginx를 선택한 이유** :
   * Apache는 다양한 MPM 방식으로 요청을 처리하며 풍부한 모듈과 유연한 설정이 장점이고, Nginx는 이벤트 기반 구조를 사용하여 다수의 동시 연결을 효율적으로 처리하는 데 강점이 있다.
  
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

### Reverse Proxy 



### 트러블슈팅

   **[이슈] `sudo nginx -t` 입력 시 응답이 지나치게 느린 현상**  
   * **확인 방법** : `time sudo nginx -t` 명령어로 측정 시 약 15초 뒤에 결과 출력
   * **원인** : 초기 서버 설정 시 호스트 이름(hostname)을 변경했으나, `/etc/hosts` 파일 내부의 이름이 동기화되지 않아 이름 조회(DNS/Local resolution) 과정에서 지연 발생
   * **해결 방법** : 서버의 호스트 이름을 변경했다면 `/etc/hosts` 파일도 함께 확인하여 일치하도록 수정해야 합니다.
  
## 2. WAS 서버

### 서버 정보

- Hostname : **was01**
- IP       : **192.168.111.142** *(예시)*
- OS       : **Ubuntu 24.04**

### WAS 서버 사전 설정

1. **Hostname 설정** :
   * 웹 서버와 구분하기 위해 호스트 이름을 `was01`로 설정합니다.
   * 현재 설정된 호스트 이름 확인 : `hostname`
   * 호스트 이름 변경 : `sudo hostnamectl set-hostname was01`

2. **시간 동기화 설정 (`timedatectl`)** :
   * 현재 시간대 확인 : `timedatectl`
   * 타임존을 서울로 변경 : `sudo timedatectl set-timezone Asia/Seoul`
   * **이유** : 장애 발생 시 시스템 로그와 애플리케이션 로그의 시간 기준을 일치시켜 정확한 원인 분석을 하기 위함입니다.

### Java (JDK 21) 설치

1. **패키지 목록 업데이트** : `sudo apt update`
2. **OpenJDK 21 설치** : `sudo apt install openjdk-21-jdk -y`
3. **버전 확인 및 역할** :
   * `java -version` : 자바로 작성된 컴파일된 프로그램(런타임 환경)을 실행합니다.
   * `javac -version` : 자바 소스 코드를 바이트코드로 컴파일합니다.

### 애플리케이션 전용 계정 생성

1. **시스템 계정 생성** :
   * 로그인 Shell이 필요 없고 홈 디렉터리가 `/opt/infra-app`인 시스템 전용 계정을 생성합니다.
   * 명령어 : `sudo useradd --system --home /opt/infra-app --shell /usr/sbin/nologin infraapp`
2. **계정 생성 확인** : `id infraapp`

### 애플리케이션 및 로그 디렉터리 구조 생성

1. **디렉터리 생성** :
   * 애플리케이션 디렉터리 : `sudo mkdir -p /opt/infra-app`
   * 로그 디렉터리 : `sudo mkdir -p /var/log/infra-app`
2. **소유권 변경** :
   * 생성한 디렉터리의 소유권을 `infraapp` 계정에 부여합니다.
   * 명령어 : `sudo chown -R infraapp:infraapp /opt/infra-app`, `sudo chown -R infraapp:infraapp /var/log/infra-app`

### Spring Boot JAR 배포 및 권한 설정

1. **JAR 파일 이동** : 빌드된 `app.jar` 파일을 WAS 서버로 이동시킨 후 디렉터리로 이동시킵니다.
   * 명령어 : `sudo mv app.jar /opt/infra-app/app.jar`
2. **소유권 설정** : 
   * 명령어 : `sudo chown infraapp:infraapp /opt/infra-app/app.jar`
   * **이유** : 스프링 애플리케이션에 보안 취약점이나 장애가 발생하더라도 `root` 권한까지 탈취당하지 않도록 권한 범위를 최소 권한 원칙에 따라 제한하기 위함입니다.
3. **배포 파일 확인** : `ls -lh /opt/infra-app/`
4. **수동 실행 테스트** : `sudo -u infraapp java -jar /opt/infra-app/app.jar`

### systemd 서비스 등록 및 자동 실행

1. **systemd 서비스 파일 생성** : 
   * 경로 : `/etc/systemd/system/infra-app.service` 파일을 생성하여 백그라운드 데몬으로 등록합니다.
2. **systemd 데몬 리로드** : 수정된 서비스 파일을 시스템이 인식하도록 반영합니다.
   * 명령어 : `sudo systemctl daemon-reload`
3. **서비스 시작** : `sudo systemctl start infra-app`
4. **서비스 상태 확인** : `sudo systemctl status infra-app`
5. **프로세스 확인** : `ps -ef | grep app.jar`
6. **헬스 체크 (HTTP 확인)** : `curl -i http://localhost:8080/health`



## 3. DB 서버 

### 서버 정보

- Hostname : **db01**
- IP       : **192.168.111.143**
- OS       : **Ubuntu 24.04**

### DB 서버 사전 설정

1. **Hostname 설정** :
   * WEB/WAS 서버와 구분하기 위해 호스트 이름을 `DB01`로 설정합니다.
   * 현재 설정된 호스트 이름 확인 : `hostname`
   * 호스트 이름 변경 : `sudo hostnamectl set-hostname db01`

2. **시간 동기화 설정 (`timedatectl`)** :
   * 현재 시간대 확인 : `timedatectl`
   * 타임존을 서울로 변경 : `sudo timedatectl set-timezone Asia/Seoul`
   * **이유** : 장애 발생 시 시스템 로그와 애플리케이션 로그의 시간 기준을 일치시켜 정확한 원인 분석을 하기 위함입니다.

### MySQL 설치
1. sudo apt update
2. sudo apt install mysql-server -y
3. mysql --version
4. sudo systemctl status mysql
5. systemctl is-enabled mysql

### 애플리케이션 SQL 스크립트 Import
1. DB01 서버로 SQL 스크립트를 로컬 PC에서 옮긴다.
2. sudo mysql < db.sql 스크립트 Import

### MySQL 원격 접속 설정
1. was01 서버에서 mysql -h 192.168.111.143 -u marble -p
2. was01에서 DB 포트까지 갈 수 있는지 테스트 : nc -zv 192.168.111.143 3306

### WAS의 Spring Boot에 DB 정보 전달
1. WAS01 서버에 DB 정보를 전달할 환경 설정 파일을 만든다. /etc/infra-app/infra-app.env
2. 파일에 DB 정보를 넣는다.
3. 최소한의 보안을 위해 소유권을 root로 바꾸고 root만 접근할 수 있게 한다.
4. sudo chmod 600 /etc/infra-app/infra-app.env
5. sudo chown root:root /etc/infra-app/infra-app.env

### systemd가 환경설정을 읽게 함
1. /etc/systemd/system/infra-app.service에 EnviromentFile=/etc/infra-app/infra-app.env 추가한다.
2. sudo systemctl daemon-reload
3. sudo systemctl restart infra-app

### Spring, MySQL 연결 확인
1. curl -i http://localhost:8080/health -> 200 OK
2. curl -i http://localhost:8080/health/db  -> 200 OK

### MySQL 로그 확인
1. sudo tail -n 30 /var/log/mysql/error.log

### DB 수동 백업

### 백업 디렉터리 만들기
1. sudo mkdir -p /backup/mysql
2. sudo chown root:root /backup/mysql
3. sudo chmod 700 /backup/mysql
4. sudo ls -ld /backup/mysql
5. 700으로 하는 이유는 DB 백업 파일에는 실제 서비스 데이터가 들어 있기 때문에 아무 사용자나 읽지 못하게 하기 위해서이다.

### MySQL 백업 실행
1. sudo sh -c 'mysqldump marble > /backup/mysql/marble_$(date +%Y%m%d_%H%M%S).sql'
2. sudo ls -lh /backup/mysql

### 백업 파일 복구 테스트
1. 임시 데이터베이스를 만들어서.
2. 위에 SQL을 임포트 한 후 테스트 한다.
3. sudo sh -c 'mysql marble_restore < /backup/mysql/marble_20261012_181530.sql'

### DB 자동 백업 스크립트
1. #!/bin/bash    
2. set -e   : 명령이 실패하면 스크립트를 멈춰라 
3. BACKUP_DIR="/backup/mysql" = 백업 경로
4. DB_NAME="marble"  : 백업할 DB이름을 변수 명에 저장
5. DATE=$(date +%Y%m%d_%H%M%S) 
6. mysqldump "$DB_NAME" > "$BACKUP_DIR/${DB_NAME}_${DATE}.sql"
7. find "$BACKUP_DIR" -type f -name "${DB_NAME}_*.sql" -mtime +7 -delete

### systemd timer 자동 백업
1. Service 파일 만들기 : sudo nano /etc/systemd/system/mysql-backup.service
2. Service 타이머 만들기 : sudo nano /etc/systemd/system/mysql-backup.timer
3. systemd timer는 기본적으로 자신과 같은 이름의 service를 예약된 시간에 시작한다.

### MySQL 장애 및 복구 테스트
1. MySQL 서비스를 의도적으로 중지
2. /health → 200 확인
3. /health/db → 503 확인
4. systemctl status mysql 확인
5. 3306 LISTEN 여부 확인
6. MySQL 서비스 복구
7. /health/db → 200 정상화 확인

### 트러블슈팅
   **[이슈] was01 서버에서 dB01 서버로 MYSQL 원격 접속 설정 시 접속 거절 됨**  
   * **확인 방법** : sudo ss -tnlp | grep :3306 해서 127.0.0.1:3306 로 결과가 나옴
   * **원인** : bind-address = 127.0.0.1 로 설정
   * **해결 방법** : /etc/mysql/mysql.conf.d/mysqld.conf에서 설정을 0.0.0.0 또는 DB IP 대역으로 열어줘야 한다. 















