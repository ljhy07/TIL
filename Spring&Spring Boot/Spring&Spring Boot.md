## 한 줄 설명

Spring Framework : Java 애플리케이션을 만들기 위한 핵심 프레임워크
Spring Boot :  Spring Framework를 더 빠르고 편하게 사용할 수 있게 해주는 실행/설정/운영 도구층

**Spring Boot는 Spring을 대체하는 것이 아니라 Spring 위에 얹힌 것**이다

## Spring Framework

Spring Framework는 Java/Kotlin 기반 애플리케이션을 만들 때 필요한 핵심 기능을 제공하는 프레임워크이다.

### 대표 기능

| 기능            | 설명                               |
| ------------- | -------------------------------- |
| IoC Container | 객체 생성과 의존성 관리를 Spring 컨테이너가 담당   |
| DI            | 필요한 객체를 직접 만들지 않고 주입받게 함         |
| AOP           | 로깅, 트랜잭션, 보안 같은 공통 관심사를 분리       |
| Transaction   | 선언적 트랜잭션 처리, 예: `@Transactional` |
| Spring MVC    | 전통적인 Servlet 기반 웹 MVC 개발         |
| WebFlux       | Reactive 웹 애플리케이션 개발             |
| Data Access   | JDBC, ORM, 트랜잭션 추상화 지원           |
| Testing       | Spring 기반 테스트 지원                 |

예를 들어 Spring Framework만 사용하면 개발자가 직접 이런 것들을 많이 설정해야한다.

```
@Configuration
@ComponentScan
@EnableWebMvc
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        // DB 연결 설정
    }

    @Bean
    public PlatformTransactionManager transactionManager() {
        // 트랜잭션 매니저 설정
    }
}
```

Spring은 강력하지만, 초기 설정이 많고 프로젝트 구성이 상대적으로 복잡할 수 있다.

## Spring Boot

Spring Boot는 Spring Framework 기반 애플리케이션을 **빠르게 만들고, 바로 실행하고, 운영하기 쉽게** 만든 도구이다.

### 핵심 기능

|기능|설명|
|---|---|
|Auto Configuration|클래스패스와 설정값을 보고 필요한 Bean을 자동 구성|
|Starter Dependency|자주 쓰는 의존성을 묶어서 제공|
|Embedded Server|Tomcat, Jetty, Netty 등을 내장해 JAR로 실행 가능|
|SpringApplication|`main()` 메서드에서 애플리케이션 실행|
|Externalized Configuration|`application.yml`, 환경변수 등으로 설정 관리|
|Actuator|헬스체크, 메트릭, 운영 모니터링 기능 제공|
|Opinionated Defaults|일반적인 선택지를 기본값으로 제공|

Spring Boot의 초기 설정이다.

```
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```

그리고 웹 의존성을 추가하면 Boot가 알아서 Spring MVC, JSON 처리, 내장 서버 등을 구성한다.

## Spring과 Spring Boot의 차이

|구분|Spring Framework|Spring Boot|
|---|---|---|
|성격|핵심 프레임워크|Spring 기반 앱을 쉽게 만드는 도구|
|목적|유연한 애플리케이션 구조 제공|빠른 개발, 쉬운 실행, 운영 편의성|
|설정|직접 설정이 많음|자동 설정 중심|
|서버|외부 WAS 배포가 일반적이었음|내장 서버로 바로 실행 가능|
|의존성 관리|개발자가 세밀하게 선택|Starter로 묶어서 제공|
|운영 기능|직접 구성 필요|Actuator 등 기본 제공|
|자유도|매우 높음|기본값이 많지만 override 가능|
|관계|기반 기술|Spring을 사용하는 방식 중 하나|

### 비유

Spring Framework는 **부품과 공구가 잘 갖춰진 개발 키트**에 가깝다.
Spring Boot는 그 키트를 바탕으로 **바로 시동 걸 수 있는 완성형 개발 환경**을 제공하는 느낌이다.

### Spring Boot가 자동으로 해주는 것

예를 들어 `spring-boot-starter-web` 또는 버전에 따른 web starter 계열 의존성을 추가하면 Boot는 대략 이런 판단을 한다.

- Spring MVC 존재 -> 웹 애플리케이션
- Jackson 존재 -> JSON 변환기를 등록
- Tomcat 존재 -> 내장 서버 띄우기
- 사용자가 직접 만든 Bean 존재 -> 기본 Bean 대신 해당 Bean 사용

이게 Spring Boot의 **Auto Configuration**이다.  
중요한 점은 “강제로 숨겨서 처리한다”가 아니라, **기본값을 제공하되 개발자가 원하면 설정이나 Bean으로 바꿀 수 있다**는 것이다.

## 결론

Spring은 핵심 엔진이고, Spring Boot는 그 엔진을 얹은 실전형 차량에 가깝다.

요즘은 대부분은 **Spring Boot로 시작**한다.
하지만 내부적으로는 여전히 Spring Framework의 IoC, DI, MVC, 트랜잭션, AOP 같은 기능을 사용한다.