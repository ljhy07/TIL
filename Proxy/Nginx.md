## Nginx란

Nginx는 클라이언트의 HTTP 요청을 받아 처리하는 고성능 네트워크 서버이다. 한 가지 용도로만 쓰이는 프로그램이 아니라 다음 역할을 수행할 수 있다.

- 웹 서버: HTML, CSS, JavaScript, 이미지 같은 정적 파일 제공
- 리버스 프록시: 요청을 Node.js, Spring Boot, Django 등의 애플리케이션 서버로 전달
- 로드 밸런서: 여러 애플리케이션 서버에 요청 분산
- HTTPS 종단점: TLS 인증서와 암호화 처리
- 콘텐츠 캐시: 백엔드 응답을 저장했다가 재사용
- TCP/UDP 프록시: HTTP가 아닌 데이터베이스·게임·메시징 프로토콜 등의 트래픽 전달
- 메일 프록시

즉, 보통 다음 위치에 배치된다.

```
사용자
  │ HTTPS 요청
  ▼
Nginx
  ├── 정적 파일 직접 반환
  ├── TLS 처리
  ├── 요청 제한 및 접근 제어
  └── 애플리케이션 서버로 전달
        ├── Spring Boot :8080
        ├── Node.js     :3000
        └── Django      :8000
```

공식적으로 Nginx는 웹 서버, 리버스 프록시, 콘텐츠 캐시, 로드 밸런서 및 TCP/UDP 프록시로 정의된다.

## 웹 서버와 리버스 프록시의 차이

### 웹 서버로 사용할 때

Nginx가 파일을 직접 읽어서 응답한다.

```
GET /images/logo.png
        ↓
Nginx가 /srv/www/images/logo.png를 읽음
        ↓
브라우저에 이미지 반환
```

정적 파일은 별도의 비즈니스 로직이 필요하지 않으므로 Nginx가 직접 처리하는 것이 효율적이다.

```
server {
    listen 80;
    server_name example.com;

    root /srv/www;

    location / {
        index index.html;
    }
}
```

### 리버스 프록시로 사용할 때

Nginx가 클라이언트 요청을 내부 애플리케이션에 전달한다.

```
브라우저 → Nginx :443 → Spring Boot :8080
```

클라이언트는 내부 서버의 주소나 포트를 알 필요가 없다.

```
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

`proxy_pass`가 리버스 프록시의 핵심 지시어이다.

참고로 “프록시”와 “리버스 프록시”는 방향이 다르다.

- 포워드 프록시: 사용자를 대신해 외부 서버에 접속
- 리버스 프록시: 내부 서버들을 대신해 외부 요청을 받음

## Nginx가 빠른 이유

Nginx의 중요한 특징은 **이벤트 기반 비동기 처리 구조**이다.

### 프로세스 구조

Nginx는 일반적으로 다음 프로세스로 구성된다.

- Master process
    - 설정 파일 읽기
    - 포트 바인딩
    - Worker 생성 및 관리
    - 설정 재적용과 로그 회전 처리
- Worker process
    - 실제 클라이언트 연결과 요청 처리

```
Master
  ├── Worker 1 ── 여러 연결 처리
  ├── Worker 2 ── 여러 연결 처리
  ├── Worker 3 ── 여러 연결 처리
  └── Worker 4 ── 여러 연결 처리
```

각 연결마다 프로세스나 스레드를 하나씩 만드는 방식이 아니라, 적은 수의 Worker가 이벤트 루프를 통해 많은 연결을 처리한다. 특히 요청 대부분이 네트워크나 디스크 I/O를 기다리는 상황에서 효율적이다. Nginx 공식 가이드에서 Worker가 이벤트 기반 모델과 운영체제별 이벤트 메커니즘을 이용한다고 설명한다.

대표적인 기본 설정은 다음과 같다.

```
worker_processes auto;

events {
    worker_connections 4096;
}
```

`worker_processes auto`는 CPU 코어 수를 기준으로 Worker 수를 자동 결정한다.

`worker_connections`는 Worker 하나가 열 수 있는 연결 수이다. 다만 단순히 다음과 같이 계산한 값이 곧 최대 사용자 수가 되는 것은 아니다.

```
worker_processes × worker_connections
```

리버스 프록시에서는 클라이언트 연결뿐 아니라 백엔드 연결도 포함하며, 운영체제의 파일 디스크립터 제한도 영향을 미친다.

## 설정 파일 구조

기본 설정 파일은 설치 방식에 따라 보통 다음 위치에 있다.

```
/etc/nginx/nginx.conf
/usr/local/nginx/conf/nginx.conf
```

Linux 패키지에서는 흔히 다음 파일들로 분리된다.

```
/etc/nginx/
├── nginx.conf
├── conf.d/
│   └── example.conf
├── sites-available/
└── sites-enabled/
```

다만 `sites-available`과 `sites-enabled` 구조는 모든 배포판의 공통 규칙이 아니라 Debian/Ubuntu 계열에서 주로 사용하는 패키징 관례이다.

설정은 여러 컨텍스트로 구성된다.

```
# main 컨텍스트
worker_processes auto;

events {
    # events 컨텍스트
    worker_connections 4096;
}

http {
    # HTTP 전체 설정

    server {
        # 하나의 가상 서버 설정

        location /api/ {
            # 특정 URI 처리 설정
        }
    }
}
```

계층은 대략 다음과 같다.

```
main
├── events
└── http
    ├── upstream
    └── server
        └── location
```

간단한 지시어는 세미콜론으로 끝난다.

```
server_name example.com;
```

블록 지시어는 중괄호를 사용한다.

```
server {
    ...
}
```

## 요청 처리

요청이 들어오면 대략 다음 순서로 처리된다.

### 1. IP와 포트 확인

```
listen 80;
listen 443 ssl;
```

### 2. Host에 맞는 `server` 선택

```
server_name example.com www.example.com;
```

### 3. URI에 맞는 `location` 선택

```
location / {
    ...
}

location /api/ {
    ...
}

location = /health {
    ...
}
```

대표적인 `location` 표현은 다음과 같다.

```
# 정확히 /health만
location = /health {
    return 200 "OK";
}

# /images/로 시작
location /images/ {
    root /srv/www;
}

# 대소문자를 구분하지 않는 정규식
location ~* \.(jpg|jpeg|png|gif)$ {
    expires 30d;
}
```

일반적으로 정확 일치가 가장 우선되고, 문자열 prefix 중에서는 더 구체적인 경로가 선택된다. 정규식 location에는 별도의 탐색 규칙이 있으므로, 복잡한 설정에서는 공식 처리 순서를 이해해야 한다.

## 상세 설정

- `upstream`: 백엔드 서버 그룹 정의
- `least_conn`: 활성 연결이 가장 적은 서버 선택
- `keepalive`: 백엔드 연결 재사용
- `proxy_pass`: 요청을 백엔드 그룹으로 전달
- `Host`: 원래 요청 도메인 전달
- `X-Real-IP`: 직접 접속한 클라이언트 주소 전달
- `X-Forwarded-For`: 프록시를 거친 IP 목록 전달
- `X-Forwarded-Proto`: 최초 요청이 HTTP인지 HTTPS인지 전달
- `proxy_*_timeout`: 연결·송신·응답 대기 시간 제한
- `client_max_body_size`: 업로드 요청 본문 크기 제한

애플리케이션에서는 프록시 헤더를 무조건 신뢰하면 안 된다. Nginx처럼 신뢰할 수 있는 프록시에서 전달된 경우에만 사용하도록 프레임워크의 trusted proxy 설정을 구성해야 한다.

## `proxy_pass` 끝의 `/`

Nginx에서 자주 생기는 문제이다.

```
location /api/ {
    proxy_pass http://backend;
}
```

이 경우 대체로 기존 URI가 유지된다.

```
/api/users → backend/api/users
```

반면:

```
location /api/ {
    proxy_pass http://backend/;
}
```

`proxy_pass`에 URI 부분인 `/`가 있으므로 일치한 `/api/` 부분이 교체된다.

```
/api/users → backend/users
```

실제 설정에서는 쿼리 문자열, 정규식 location, 변수 사용 여부에 따라 동작이 더 복잡해질 수 있으므로 경로 변환이 필요하면 반드시 테스트하는 것이 좋다.

## 로드 밸런싱

여러 백엔드 서버가 있을 때 Nginx가 요청을 나눌 수 있다.

### Round Robin

기본 방식이다.

```
upstream backend {
    server app1:8080;
    server app2:8080;
    server app3:8080;
}
```

요청을 서버들에 순환 분배한다.

### Least Connections

```
upstream backend {
    least_conn;

    server app1:8080;
    server app2:8080;
}
```

현재 연결이 적은 서버에 요청을 보냅니다. 요청 처리 시간이 서로 크게 다를 때 유용할 수 있다.

### 가중치

```
upstream backend {
    server app1:8080 weight=3;
    server app2:8080 weight=1;
}
```

`app1`에 더 많은 요청을 할당한다.

### IP Hash

```
upstream backend {
    ip_hash;

    server app1:8080;
    server app2:8080;
}
```

클라이언트 IP를 기반으로 서버를 선택해 같은 사용자가 가능한 한 같은 백엔드로 전달되게 한다.

다만 세션을 서버 메모리에 저장한 뒤 `ip_hash`에 의존하는 것보다는 세션을 Redis나 데이터베이스 같은 외부 저장소에 두어 애플리케이션 서버를 무상태로 만드는 편이 일반적으로 확장과 장애 대응에 유리한다.

Nginx 오픈소스 버전은 기본적인 수동적 장애 감지를 지원한다. 고급 능동 상태 점검이나 동적 재구성 등의 일부 기능은 NGINX Plus에 포함될 수 있다.

## HTTPS 처리

Nginx에서 TLS 인증서를 설정하면 애플리케이션 서버는 내부에서 HTTP만 사용할 수도 있다.

```
server {
    listen 80;
    server_name example.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

이 구조를 **TLS termination**이라고 한다.

```
사용자 ── HTTPS ──> Nginx ── HTTP 또는 HTTPS ──> 애플리케이션
```

내부 구간도 신뢰할 수 없는 네트워크를 통과한다면 Nginx와 백엔드 사이에도 TLS 또는 mTLS를 적용해야 한다. 

## WebSocket 프록시

WebSocket은 HTTP 연결 업그레이드 헤더를 전달해야 한다.

```
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        listen 80;

        location /socket/ {
            proxy_pass http://127.0.0.1:3000;

            proxy_http_version 1.1;
            proxy_set_header Upgrade    $http_upgrade;
            proxy_set_header Connection $connection_upgrade;

            proxy_read_timeout 60s;
        }
    }
}
```

WebSocket뿐 아니라 Server-Sent Events, 스트리밍 응답, 대용량 업로드에서는 버퍼링과 타임아웃 설정도 별도로 검토해야 한다.

## 캐싱

Nginx가 백엔드의 응답을 캐싱하면 같은 요청마다 애플리케이션을 호출하지 않아도 된다.

```
proxy_cache_path /var/cache/nginx
                 levels=1:2
                 keys_zone=api_cache:10m
                 max_size=1g
                 inactive=60m;

server {
    location /products/ {
        proxy_pass http://backend;

        proxy_cache api_cache;
        proxy_cache_valid 200 10m;

        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

예를 들어 상품 목록처럼 자주 조회되지만 자주 변경되지 않는 응답에 효과적이다.

하지만 다음 응답은 특별히 조심해야 한다.

- 로그인한 사용자마다 달라지는 응답
- `Authorization` 헤더가 있는 요청
- 세션 쿠키에 의존하는 응답
- 개인정보가 포함된 응답
- 실시간성이 중요한 데이터

캐시 키를 잘못 설계하면 한 사용자의 응답이 다른 사용자에게 전달되는 보안 사고가 발생할 수 있다. Nginx는 응답의 `Cache-Control`, `Expires`, `Set-Cookie`, `Vary` 등도 캐시 여부 판단에 사용한다. 

## 요청 제한

과도한 API 호출을 제한할 수 있다.

```
http {
    limit_req_zone $binary_remote_addr
                   zone=per_ip:10m
                   rate=10r/s;

    server {
        location /api/ {
            limit_req zone=per_ip burst=20 nodelay;
            proxy_pass http://backend;
        }
    }
}
```

의미는 다음과 같다.

- IP별 평균 초당 10개 요청 허용
- 순간적으로 최대 20개까지 버스트 허용
- 허용량을 넘는 요청은 지연시키지 않고 즉시 제한

단, Nginx가 CDN이나 다른 프록시 뒤에 있다면 $remote_addr가 실제 사용자 IP가 아니라 앞단 프록시 IP일 수 있다. 이 경우 `real_ip` 관련 설정과 신뢰할 수 있는 프록시 범위 구성이 먼저 필요하다.

또한 Rate Limit는 기본적인 보호 장치이지 완전한 DDoS 방어 시스템은 아니다. 대규모 공격은 CDN, 클라우드 방화벽 또는 전용 DDoS 보호 계층에서 먼저 흡수해야 한다.

## 무중단 설정 재적용

설정을 수정한 후에는 먼저 문법을 검사한다.

```
sudo nginx -t
```

성공한 경우 재적용한다.

```
sudo nginx -s reload
```

systemd 환경이라면 다음 명령을 쓰기도 한다.

```
sudo systemctl reload nginx
```

Reload가 이루어지면 Master는 새 설정을 확인하고 새 Worker를 시작한다. 기존 Worker는 새 연결을 받지 않으면서 처리 중인 요청을 마무리한 뒤 종료한다. 새 설정 적용에 실패하면 기존 설정과 Worker를 유지한다. 이것이 Nginx가 무중단에 가깝게 설정을 변경할 수 있는 원리이다.

## 주요 로그와 점검 명령

기본 로그 위치는 설치 방식에 따라 다르지만 보통 다음과 같다.

```
/var/log/nginx/access.log
/var/log/nginx/error.log
```

자주 쓰는 명령어이다.

```
# 버전 확인
nginx -v

# 컴파일 옵션과 포함 모듈 확인
nginx -V

# 설정 문법 및 참조 파일 확인
sudo nginx -t

# 실제 적용될 전체 설정 출력
sudo nginx -T

# 설정 재적용
sudo nginx -s reload

# 정상 종료
sudo nginx -s quit

# 즉시 종료
sudo nginx -s stop
```

장애가 발생했을 때는 보통 다음 순서로 확인한다.

```
nginx -t
  ↓
error.log
  ↓
백엔드에 직접 curl
  ↓
Nginx를 통해 curl
  ↓
포트·방화벽·권한·DNS 확인
```

대표적인 상태 코드는 다음과 관련이 있다.

- `400`: 잘못된 요청, 헤더 또는 프로토콜 문제
- `403`: 파일 권한이나 접근 제어 문제
- `404`: `root`, `alias`, URI 매핑 문제
- `413`: `client_max_body_size` 초과
- `499`: 클라이언트가 응답 전에 연결 종료한 것을 Nginx가 기록
- `502`: 백엔드 접속 실패 또는 잘못된 응답
- `504`: 백엔드 응답 시간 초과

## Nginx의 애플리케이션 서버 대체 여부

대부분의 경우 그렇지 않다.

Nginx는 HTTP 전달, 정적 파일, 캐싱, TLS 처리 등에 강하지만 다음과 같은 애플리케이션 로직을 일반적으로 대신하지 않는다.

- 주문 처리
- 회원 가입
- 데이터베이스 접근
- 결제 로직
- 복잡한 인증·인가
- 도메인 규칙 실행

일반적인 역할 분리는 다음과 같다.

```
Nginx
- HTTPS
- 정적 파일
- 요청 라우팅
- 캐시
- 요청 제한
- 로드 밸런싱

Spring Boot / Node.js / Django 등
- 비즈니스 로직
- 데이터베이스
- 사용자 인증
- API 응답 생성
```

PHP의 경우에는 Nginx가 PHP 코드를 직접 실행하지 않고 PHP-FPM에 FastCGI로 전달한다.

## 장점과 단점

### 장점

- 많은 동시 연결을 효율적으로 처리
- 정적 파일 제공 성능이 좋음
- 리버스 프록시와 로드 밸런싱 설정이 비교적 간결함
- 설정 재적용 시 연결 중단을 줄일 수 있음
- TLS, 캐시, 압축, 접근 제어 등을 한 계층에서 처리 가능
- 운영 사례와 문서가 풍부함

### 단점과 주의점

- 설정 문법이 프로그래밍 언어처럼 직관적이지 않을 수 있음
- `location`, `root`, `alias`, `rewrite`, `proxy_pass` 조합은 실수하기 쉬움
- 설정 변경이 자동 반영되지 않으므로 검사와 reload가 필요함
- 일부 고급 기능은 NGINX Plus에서만 제공
- 동적 서비스 디스커버리와 복잡한 L7 정책은 Envoy·Traefik 등의 도구가 더 편한 경우도 있음
- 인증·인가를 전부 Nginx 설정에 넣으면 유지보수가 어려워질 수 있음

## 다른 서버와의 대략적인 차이

- Apache HTTP Server: 전통적인 웹 호스팅 생태계, `.htaccess`, 모듈 호환성이 중요한 환경에서 강점
- Caddy: 자동 HTTPS와 간단한 설정이 강점
- HAProxy: 고성능 프록시와 로드 밸런싱에 특화
- Traefik: Docker·Kubernetes의 동적 서비스 검색과 연동이 편리
- Envoy: 서비스 메시, 관측성, 복잡한 네트워크 정책에 강함
- Nginx: 정적 파일, 리버스 프록시, TLS, 캐시, 로드 밸런싱을 균형 있게 제공

Nginx는 “웹 서버”라고만 이해하기보다 **외부 트래픽과 내부 애플리케이션 사이에서 요청을 통제하고 전달하는 앞단 네트워크 계층**으로 이해하면 가장 정확하다.