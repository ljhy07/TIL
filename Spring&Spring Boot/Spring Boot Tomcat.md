## Spring, Spring Boot, Tomcat

세 가지는 함께 사용되지만 역할은 다르다.

| 구분               | 역할                        | 대표적인 기능                      |
| ---------------- | ------------------------- | ---------------------------- |
| Spring Framework | 애플리케이션 개발의 기반             | 객체 관리, 의존성 주입, 트랜잭션, 웹 요청 처리 |
| Spring Boot      | Spring 애플리케이션의 설정과 실행 간소화 | 자동 설정, 의존성 관리, 내장 서버 구성      |
| Apache Tomcat    | Java 웹 애플리케이션 실행 환경       | HTTP 통신, 서블릿 실행, 연결과 요청 처리   |

예를 들어 사용자가 상품 목록을 조회한다면 다음처럼 역할이 나뉜다.

- **Tomcat:** HTTP 요청을 받아 Java 코드가 처리할 수 있는 형태로 전달한다.
- **Spring MVC:** 요청 URL에 맞는 컨트롤러 메서드를 찾고 호출한다.
- **애플리케이션 코드:** 상품을 조회하고 응답 데이터를 만든다.
- **Spring Boot:** 이 구성 요소들이 함께 동작하도록 설정을 자동화한다.

Spring Boot 자체가 HTTP 통신을 모두 처리하는 것은 아니다. 서블릿 기반 웹 애플리케이션에서는 기본적으로 내장 Tomcat을 사용하며, 다른 서버로 교체할 수도 있다.

## Tomcat은 정확히 어떤 서버인가

Tomcat은 **HTTP 서버 기능을 가진 서블릿 컨테이너**이다.

여기서 두 가지 개념을 구분하면 이해하기 쉽다.

### HTTP 서버 기능

클라이언트와 HTTP로 통신하는 기능이다.

예를 들어 다음 요청을 수신하고 응답을 돌려준다.

```
GET /products/42 HTTP/1.1
Host: localhost:8080
```

Tomcat은 네트워크 연결을 받고, 요청을 해석하고, 응답을 클라이언트에게 전송한다.

### 서블릿 컨테이너 기능

서블릿은 Java에서 웹 요청과 응답을 처리하기 위한 표준 기반 구성 요소이다. Tomcat은 서블릿을 생성하고 초기화하며, 요청을 전달하고, 종료 시 정리하는 실행 환경을 제공한다.

대표적인 생명주기는 다음과 같다.

```
init()       → 초기화
service()    → 요청 처리
destroy()    → 종료 시 정리
```

일반적인 구성에서는 서블릿 인스턴스 하나에 여러 요청이 동시에 들어올 수 있다. 요청마다 서블릿 객체를 새로 만드는 방식은 아니다.

Tomcat을 WAS라고 부르기도 하지만, 더 정확하게는 **서블릿 컨테이너**이다. 전체 Jakarta EE 기능을 모두 제공하는 서버와는 지원 범위가 다르다.

## 내장 Tomcat

내장 Tomcat은 **Tomcat을 애플리케이션의 라이브러리로 포함하고, 애플리케이션 실행 과정에서 함께 시작하는 방식**이다.

Spring Boot 3.x에서 다음 의존성을 추가하면 기본적으로 Tomcat도 함께 포함된다.

```
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
}
```

의존성 관계를 단순화하면 다음과 같다.

```
spring-boot-starter-web
├── Spring MVC 관련 라이브러리
├── JSON 처리 관련 라이브러리
└── spring-boot-starter-tomcat
    └── 내장 Tomcat 관련 라이브러리
```

따라서 Tomcat을 별도 설치하지 않아도 실행 가능한 JAR을 다음처럼 시작할 수 있다.

```
java -jar app.jar
```

**애플리케이션과 내장 Tomcat은 같은 JVM 프로세스 안에서 동작한다.** Tomcat이 별도 프로세스로 자동 실행되는 구조는 아니다.

##  애플리케이션 실행 시 동작

보통 다음 코드에서 시작한다.

```
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

실제 초기화 단계는 서로 연결되어 있지만, 개념적으로는 다음 흐름이다.

1. Spring Boot가 설정과 의존성을 확인한다.
2. 서블릿 기반 웹 애플리케이션에 필요한 Spring 컨텍스트를 구성한다.
3. 자동 설정을 통해 내장 Tomcat을 생성하고 설정한다.
4. Spring MVC의 `DispatcherServlet` 등을 등록한다.
5. 서버가 지정된 포트에서 요청을 받을 준비를 마친다.

이때 **Spring 컨테이너**와 **서블릿 컨테이너**는 서로 다른 개념이다.

|컨테이너|관리 대상|
|---|---|
|Spring 컨테이너|컨트롤러, 서비스, 리포지토리 등 Spring 빈|
|서블릿 컨테이너인 Tomcat|서블릿 실행 환경, 필터 체인, 웹 요청과 응답 등|

두 컨테이너가 연동되기 때문에 Tomcat으로 들어온 요청이 Spring의 컨트롤러까지 전달된다.

## HTTP 요청이 컨트롤러에 도착하는 과정

다음 API가 있다고 가정해 본다.

```
@RestController
public class HelloController {

    @GetMapping("/hello")
    public Map<String, String> hello() {
        return Map.of("message", "Hello");
    }
}
```

사용자가 `GET /hello`를 요청하면 다음 순서로 처리된다.

```
브라우저 / 모바일 앱
        ↓ HTTP 요청
Tomcat Connector
        ↓
서블릿 필터 체인
        ↓
DispatcherServlet
        ↓
HandlerMapping: 실행할 핸들러 탐색
        ↓
HandlerAdapter: 컨트롤러 호출 지원
        ↓
HelloController.hello()
        ↓
응답 객체를 JSON으로 변환
        ↓
Tomcat을 통해 HTTP 응답 전송
```

여기서 중요한 구성 요소는 `DispatcherServlet`이다.

`DispatcherServlet`은 Spring MVC가 제공하는 서블릿으로, 웹 요청 처리를 조정하는 중앙 진입점이다. 요청에 맞는 핸들러를 찾고, 실행을 위임하고, 응답 처리와 예외 처리에 필요한 구성 요소를 연결한다.

역할을 구체적으로 구분하면 다음과 같다.

|작업|주로 담당하는 구성 요소|
|---|---|
|네트워크 연결과 HTTP 요청 수신|Tomcat|
|서블릿 필터 실행|Tomcat의 필터 체인|
|URL에 맞는 컨트롤러 탐색|Spring MVC|
|비즈니스 로직 수행|컨트롤러·서비스 등 애플리케이션 코드|
|객체를 JSON으로 변환|Spring의 메시지 변환기와 JSON 라이브러리|
|HTTP 응답을 네트워크로 전송|Tomcat|

따라서 **Tomcat이 `@GetMapping`을 직접 해석해서 컨트롤러를 호출하는 것은 아니다.**

## Tomcat 내부 구조: Connector와 Container

Tomcat 내부를 이해할 때는 먼저 **Connector와 Container**를 구분하면 된다.

- **Connector:** 외부 연결을 받고 HTTP 프로토콜을 처리한다.
- **Container:** 요청을 적절한 웹 애플리케이션과 서블릿으로 전달한다.

전체 구조는 대략 다음과 같다.

```
Server
└── Service
    ├── Connector
    └── Engine
        └── Host
            └── Context
                └── Wrapper
                    └── Servlet
```

|구성 요소|의미|
|---|---|
|Server|Tomcat 서버 전체|
|Service|Connector와 Engine을 연결하는 구성 단위|
|Connector|HTTP 등의 연결과 프로토콜 처리|
|Engine|요청을 적절한 Host로 전달|
|Host|가상 호스트|
|Context|하나의 웹 애플리케이션|
|Wrapper|개별 서블릿을 감싸는 구성 요소|

여기서 **Tomcat의 `Context`와 Spring의 `ApplicationContext`는 서로 다른 개념**이다. 전자는 웹 애플리케이션 단위이고, 후자는 Spring 빈을 관리하는 컨테이너이다.

## Tomcat의 여러 요청 동시 처리

일반적인 플랫폼 스레드 기반 Spring MVC 환경에서는 Tomcat의 작업 스레드가 요청을 처리한다.

```
요청 A → 작업 스레드 1 → Controller → Service → DB
요청 B → 작업 스레드 2 → Controller → Service → DB
요청 C → 작업 스레드 3 → Controller → Service → 외부 API
```

동기식 요청 처리에서는 DB나 외부 API의 응답을 기다리는 동안에도 해당 요청의 작업 스레드가 점유될 수 있다.

이때 다음 두 가지를 구분해야 한다.

- **연결 수:** 서버가 유지하는 네트워크 연결 개수
- **작업 스레드 수:** 요청 처리를 수행할 수 있는 스레드 개수

Tomcat의 NIO 커넥터는 네트워크 연결을 관리하므로 **연결 하나가 항상 작업 스레드 하나를 계속 차지하는 것은 아니다.** 다만 NIO를 쓴다고 동기식 JDBC 호출까지 자동으로 논블로킹이 되는 것은 아니다.

또한 컨트롤러와 서비스는 보통 싱글턴 빈이다. 여러 요청이 같은 객체를 함께 사용하므로 요청별 상태를 인스턴스 필드에 저장하면 문제가 생길 수 있다.

```
@RestController
public class UnsafeController {

    private String currentUser; // 여러 요청이 공유하는 상태
}
```

사용자별 데이터는 메서드 지역 변수나 적절한 요청·세션 범위의 저장 방식으로 다루는 것이 좋다.

## Tomcat 설정

|설정|의미|
|---|---|
|`server.port`|서버가 요청을 받는 포트|
|`server.servlet.context-path`|애플리케이션 URL의 기본 경로|
|`server.tomcat.threads.max`|플랫폼 작업 스레드의 최대 개수|
|`server.tomcat.threads.min-spare`|유지할 최소 작업 스레드 수|
|`server.tomcat.max-connections`|동시에 수용하는 연결 수의 상한|
|`server.tomcat.accept-count`|연결 수 상한 도달 시 OS에 요청하는 연결 대기열 크기|
|`server.tomcat.connection-timeout`|연결 수락 후 요청 URI 라인이 도착하기를 기다리는 시간|

위 설정과 앞의 컨트롤러를 함께 사용하면 요청 경로는 `/api/hello`가 된다. 설정의 지원 여부와 기본값은 사용하는 버전 문서를 기준으로 확인해야 한다.

특히 다음 두 가지를 자주 혼동한다.

**`connection-timeout`은 API 전체 실행 제한 시간이 아니다.**

이 값을 `20s`로 설정해도 컨트롤러에서 실행 중인 긴 DB 쿼리가 20초 후 자동 중단되는 것은 아니다. DB 쿼리 타임아웃, 외부 HTTP 클라이언트 타임아웃 등은 별도로 설정해야 한다.

**`accept-count`는 단순히 “스레드가 부족할 때 대기하는 HTTP 요청 수”가 아니다.**

연결 수 상한과 OS의 연결 대기열에 관련된 설정이다. 실제 대기열 동작에는 운영체제도 영향을 준다. 가상 스레드나 별도 Executor를 쓰면 스레드 설정의 적용 방식도 달라진다.

## 스레드를 늘리면 성능이 좋아지는가

**병목이 어디에 있느냐에 따라 다르다.**

예를 들어 다음 상황을 생각해 볼 수 있다.

```
Tomcat 작업 스레드: 200개
DB 커넥션 풀: 20개
대부분의 요청: DB 접근 필요
```

DB 커넥션이 모두 사용 중이면 나머지 요청은 커넥션 반환을 기다릴 수 있다. 이때 Tomcat 스레드를 더 늘려도 DB 처리 능력이 그대로라면 처리량 개선은 제한적이다.

오히려 다음 비용이 커질 수 있다.

- 스레드가 사용하는 메모리
- 스레드 간 실행 전환 비용
- DB 연결을 기다리는 요청 수
- 과부하 시 응답 지연

성능을 판단할 때는 Tomcat만 보지 않고 **CPU 사용률, 요청 지연, 활성 스레드, DB 커넥션 대기, 쿼리 시간, 외부 API 응답 시간**을 함께 봐야 한다.

예를 들어 초당 100건의 요청이 들어오고 평균 처리 시간이 0.2초라면, 안정적인 상태에서 처리 중인 요청은 평균적으로 약 20건이다.

```
평균 처리 중 요청 수 ≈ 초당 처리량 × 평균 처리 시간
                    ≈ 100 × 0.2
                    ≈ 20
```

이는 용량을 이해하는 출발점일 뿐이다. 실제 설정에는 순간적인 트래픽 증가와 느린 요청도 반영해야 한다.

## 내장 Tomcat과 외장 Tomcat의 배포 차이

**내장 Tomcat 방식**

애플리케이션에 서버 라이브러리를 포함하고 직접 실행한다.

```
실행 가능한 JAR
├── 애플리케이션 코드
├── Spring 라이브러리
└── Tomcat 라이브러리
```

```
java -jar app.jar
```

**외장 Tomcat 방식**

Tomcat을 별도로 설치하고 애플리케이션을 WAR 파일로 배포한다.

```
별도로 설치한 Tomcat
└── 배포된 애플리케이션 WAR
```

|항목|내장 Tomcat|외장 Tomcat|
|---|---|---|
|일반적인 배포 파일|실행 가능한 JAR|WAR|
|시작 주체|애플리케이션|Tomcat 서버|
|서버 버전 관리|애플리케이션 의존성으로 관리|설치된 서버 기준으로 관리|
|설정 중심|Spring Boot 설정|외장 서버 설정과 애플리케이션 설정|
|실행 단위|보통 애플리케이션별 JVM|한 JVM에 여러 앱 배포 가능|

Spring Boot의 전통적인 WAR 배포에서는 보통 `SpringBootServletInitializer`를 사용하고, Tomcat 의존성을 제공받는 범위로 지정한다. 외장 서버의 포트나 Connector 설정은 외장 Tomcat 쪽에서 관리해야 한다.

## Nginx가 있어도 Tomcat이 필요한가

Spring MVC 애플리케이션에서는 다음처럼 함께 배치할 수 있다.

```
클라이언트
    ↓
Nginx / 로드 밸런서
    ↓
Spring Boot + Tomcat
    ↓
데이터베이스
```

앞단의 프록시는 HTTPS 종료, 요청 분산, 정적 파일 제공 등을 맡을 수 있고, Tomcat은 Java 웹 애플리케이션 실행을 담당한다.

**Tomcat 자체도 HTTP 요청을 받을 수 있으므로 Nginx가 반드시 필요한 것은 아니다.** 서비스의 배포 구조에 따라 앞단 프록시나 로드 밸런서를 추가한다.

##  서버 종료 시 진행 중인 요청 처리

배포나 재시작 시에는 진행 중인 요청이 마무리될 시간을 주는 **Graceful Shutdown**을 사용할 수 있다.

```
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

정상적인 종료 신호를 받으면 서버가 새로운 요청 수락을 중단하고, 진행 중인 요청이 종료되기를 제한된 시간 동안 기다린다.

`30s`는 종료 단계에 허용하는 시간이며, 모든 요청이 반드시 성공한다는 보장은 아니다. 강제 종료에서는 이 절차가 실행되지 않을 수 있다. 운영 환경의 종료 유예 시간도 함께 맞춰야 한다.

## 자주 헷갈리는 내용

| 질문                             | 답변                                                             |
| ------------------------------ | -------------------------------------------------------------- |
| Spring Boot에는 항상 Tomcat이 필요한가? | 아니다. 배치·CLI 애플리케이션은 웹 서버 없이 실행할 수 있다.                          |
| Spring MVC는 Tomcat에서만 실행되는가?   | 아니다. 호환되는 다른 서블릿 컨테이너에서도 실행할 수 있다.                             |
| `@RestController` 자체가 서블릿인가?   | 아니다. Spring MVC가 호출하는 Spring 빈이며, 앞단에 `DispatcherServlet`이 있다. |
| 요청마다 컨트롤러 객체가 만들어지는가?          | 기본적으로 아니다. 보통 하나의 싱글턴 객체를 여러 요청이 사용한다.                         |
| 내장 Tomcat은 개발용인가?              | 아니다. 운영 배포에도 사용할 수 있다.                                         |
| WebFlux도 항상 Tomcat을 쓰는가?       | 아니다. 일반적인 WebFlux 스타터 구성은 Reactor Netty를 기본으로 사용한다.            |

실제 요청을 추적할 때는 **“Tomcat이 HTTP 요청을 받고 → `DispatcherServlet`이 Spring MVC 처리를 조정하고 → 컨트롤러와 서비스가 기능을 실행한다”**는 경계를 기준으로 보면 각 기술의 역할이 명확해진다.