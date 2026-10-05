# Phase 1. On-Premise 3-Tier 구축

## 1. 구성

```text
Client
  ↓
WEB01
192.168.111.141
Nginx :80
  ↓
WAS01
192.168.111.142
JDK 17 + Tomcat 10
  ↓
DB01
192.168.111.143
MySQL :3306
```

### 서버 정보

| 서버 | IP | 구성 |
|---|---|---|
| WEB01 | 192.168.111.141 | Nginx |
| WAS01 | 192.168.111.142 | JDK 17, Tomcat 10, WAR |
| DB01 | 192.168.111.143 | MySQL |

---

# 2. WEB Server

## Nginx 설치

패키지 관리 및 업데이트 편의성을 위해 APT 패키지로 설치했다.

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
```

### 확인

```bash
sudo systemctl status nginx
sudo ss -tnlp | grep :80
```

Ubuntu 패키지 설치 시 Nginx Worker는 기본적으로 `www-data` 계정을 사용하므로 별도의 웹 전용 계정은 생성하지 않았다.

### WAS Reverse Proxy 설정

`/etc/nginx/sites-available/default`

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://192.168.111.142:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

설정 확인 및 반영:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

# 3. WAS Server

## JDK 17 설치

```bash
sudo apt update
sudo apt install openjdk-17-jdk -y
```

확인:

```bash
java -version
javac -version
```

## Tomcat 10 설치

JSP 기반 WAR 애플리케이션을 외부 Tomcat에 배포하기 위해 Tomcat 10을 사용했다.

```bash
sudo apt install tomcat10 -y
sudo systemctl enable tomcat10
```

확인:

```bash
sudo systemctl status tomcat10
sudo ss -tnlp | grep :8080
```

### 주요 경로

```text
CATALINA_HOME : /usr/share/tomcat10
CATALINA_BASE : /var/lib/tomcat10

WAR 배포 :
/var/lib/tomcat10/webapps/

로그 :
/var/log/tomcat10
```

## WAR 배포

IntelliJ/Gradle에서 `ROOT.war` 생성 후 WAS 서버로 전달했다.

```bash
sudo cp ROOT.war /var/lib/tomcat10/webapps/
sudo systemctl restart tomcat10
```

로그 확인:

```bash
sudo journalctl -u tomcat10 -f
```

Tomcat 로그에서 다음 내용을 확인했다.

```text
Apache Tomcat/10.1.55
Java 17
Deploying web application archive [ROOT.war]
Deployment ... has finished
Starting ProtocolHandler ["http-nio-8080"]
```

따라서 WAR 배포 및 Tomcat 8080 기동이 정상임을 확인했다.

---

# 4. DB Server

## MySQL 설치

```bash
sudo apt update
sudo apt install -y mysql-server
```

확인:

```bash
mysql --version
sudo systemctl status mysql
sudo systemctl enable mysql
```

## Database 생성

```sql
CREATE DATABASE shopmall
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

## WAS 전용 계정 생성

WAS01에서만 DB에 접근하도록 계정을 제한했다.

```sql
CREATE USER 'shopadmin'@'192.168.111.142'
IDENTIFIED BY '비밀번호';

GRANT ALL PRIVILEGES ON shopmall.*
TO 'shopadmin'@'192.168.111.142';

FLUSH PRIVILEGES;
```

## 원격 접속 허용

기본 설정에서는 MySQL이 localhost만 Listen하고 있어 WAS에서 접속할 수 없었다.

`/etc/mysql/mysql.conf.d/mysqld.cnf`

```text
bind-address = 127.0.0.1
```

을 다음과 같이 변경했다.

```text
bind-address = 0.0.0.0
```

적용:

```bash
sudo systemctl restart mysql
```

확인:

```bash
sudo ss -tnlp | grep :3306
```

---

# 5. WAS → DB 연결 확인

WAS01에서:

```bash
nc -zv 192.168.111.143 3306
```

결과:

```text
Connection to 192.168.111.143 3306 port [tcp/mysql] succeeded!
```

MySQL 실제 로그인:

```bash
mysql -h 192.168.111.143 -u shopadmin -p
```

정상 접속을 확인했다.

Spring 설정:

```properties
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://192.168.111.143:3306/shopmall?serverTimezone=Asia/Seoul
spring.datasource.username=shopadmin
spring.datasource.password=${DB_PASSWORD}
```

---

# 6. Troubleshooting

## ① WAS → DB 3306 Connection refused

### 증상

```text
nc: connect to 192.168.111.143 port 3306 failed:
Connection refused
```

### 원인

MySQL이 `127.0.0.1:3306`에서만 Listen하고 있어 외부 WAS 서버의 연결을 받을 수 없었다.

### 해결

```text
bind-address = 0.0.0.0
```

으로 변경하고 MySQL을 재시작했다.

---

## ② DB 서버에서 shopadmin 로그인 실패

### 증상

```text
Host '192.168.111.143' is not allowed to connect
```

### 원인

`shopadmin` 계정을 다음과 같이 생성했기 때문이다.

```text
shopadmin@192.168.111.142
```

즉 WAS01에서만 접근하도록 제한한 계정이었다.

### 결과

DB 서버 로컬에서 접근이 차단되는 것은 정상이며 WAS01에서는 정상 접속했다.

---

## ③ WAS에서 Access denied 발생

### 증상

```text
Access denied for user
'shopadmin'@'192.168.111.142'
```

### 원인

MySQL 계정의 인증 정보 확인이 필요했다.

### 조치

계정 Host, 비밀번호 및 권한을 확인한 뒤 WAS01에서 정상 로그인되는 것을 확인했다.

---

# 7. 최종 확인

최종적으로 다음 흐름의 통신을 확인했다.

```text
Browser
   ↓
WEB01 :80
Nginx
   ↓
WAS01 :8080
Tomcat + WAR
   ↓
DB01 :3306
MySQL
```

WEB → WAS → DB까지 정상 연결되어 On-Premise 3-Tier 환경 구축을 완료했다.
