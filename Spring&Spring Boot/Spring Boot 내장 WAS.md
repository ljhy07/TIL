## Spring Boot의 **내장 WAS**
애플리케이션 안에 Tomcat 같은 웹 서버를 포함해서, 별도의 WAS에 배포하지 않고도 실행할 수 있게 해주는 구조이다.

Spring
```
WAR 파일 생성 → 외부 Tomcat/WebLogic/JBoss 등에 배포 → WAS가 애플리케이션 실행
```

Spring Boot
```
JAR 파일 실행 → 애플리케이션 안의 내장 Tomcat 실행 → 서버 시작
```

## **기본 구조**

Spring Boot에서 `spring-boot-starter-web`을 사용하면 기본적으로 **내장 Tomcat**이 포함된다.

```
@SpringBootApplication
public class App {
    public static void main(String[] args) {
        SpringApplication.run(App.class, args);
    }
}
```

위 코드 실행 시 Spring Boot의 동작 순서

1. Spring 애플리케이션 컨텍스트 생성
2. 내장 Tomcat 실행
3. DispatcherServlet 등록
4. 지정된 포트에서 HTTP 요청 수신

기본 포트는 `8080`이다.

```
server:
  port: 8081
  servlet:
    context-path: /api
```

## **쓰는 이유**

내장 WAS의 가장 큰 장점은 배포가 단순해진다는 점이다.

```
java -jar my-app.jar
```

이 한 줄이면 서버가 뜬다. 그래서 Docker, Kubernetes, 클라우드 환경과 잘 맞는다. 애플리케이션마다 독립적인 서버 설정을 가질 수 있고, 운영 환경에서 “Tomcat 버전이 서버마다 다르다” 같은 문제도 줄어든다.

## **지원하는 내장 서버**

Spring Boot MVC 기반에서는 보통 다음 서버를 사용할 수 있다.

- **Tomcat**: 기본값, 가장 흔함
- **Jetty**: 가볍고 오래된 생태계
- **Undertow**: 고성능 비동기 처리에 강점

Spring WebFlux를 쓰면 기본적으로 **Reactor Netty**가 사용된다.

## **외장 WAS와의 차이**

외장 WAS 방식은 하나의 WAS 위에 여러 애플리케이션을 올리는 방식이다.

```
Tomcat
 ├─ app1.war
 ├─ app2.war
 └─ app3.war
```

내장 WAS 방식은 애플리케이션마다 서버가 포함된다.

```
app1.jar + Tomcat
app2.jar + Tomcat
app3.jar + Tomcat
```

요즘 Spring Boot 서비스는 대부분 후자의 방식을 쓴다.

## **주의할 점**

내장 WAS를 쓴다고 해서 운영 설정이 필요 없는 것은 아니다. 운영에서는 보통 다음 설정을 신경 쓴다.

```
server:
  port: 8080
  shutdown: graceful
  tomcat:
    threads:
      max: 200
    max-connections: 8192
```

또한 앞단에 Nginx, ALB, API Gateway 같은 리버스 프록시를 두는 경우가 많다.

## 정리
Spring Boot 내장 WAS는 **애플리케이션이 자기 실행 환경을 직접 포함하는 방식**이다. 덕분에 별도 WAS 설치와 배포 과정이 줄어들고, `java -jar`만으로 실행 가능한 독립적인 서버 애플리케이션을 만들 수 있다.