# Spring Boot AutoConfiguration

## 한 문장 정리

**AutoConfiguration은 현재 프로젝트의 의존성, 설정값, 이미 등록된 Bean 등을 확인해서 필요한 Spring Bean과 기본 설정을 자동으로 등록해 주는 기능이다.**

예를 들어 프로젝트에 다음 요소가 있다고 하면

- Spring MVC 관련 의존성
- Jackson
- 내장 웹 서버
- `@SpringBootApplication`

그러면 Spring Boot는 조건을 확인한 후 다음과 같은 웹 애플리케이션 기반 설정을 자동으로 구성한다.

- `DispatcherServlet`
- JSON 메시지 변환기
- Spring MVC 기반 설정
- 내장 웹 서버
- 오류 처리와 정적 리소스 처리

개발자는 이런 인프라 설정을 일일이 작성하지 않고 Controller와 비즈니스 로직부터 만들 수 있다.

중요한 점은 **무조건 등록하는 것이 아니라, 조건이 맞을 때만 등록한다는 것**이다. Spring Boot 공식 설명도 AutoConfiguration을 “추가된 JAR 의존성을 바탕으로 애플리케이션을 구성하는 기능”으로 정의한다.

---

## AutoConfiguration이 필요한 이유

순수 Spring으로 웹 애플리케이션을 만든다면 개발자가 직접 다음과 같은 설정을 작성해야 한다.

```
@Configuration
public class WebConfig {

    @Bean
    public SomeWebInfrastructureBean webInfrastructureBean() {
        return new SomeWebInfrastructureBean();
    }
}
```

데이터베이스를 사용할 때도 직접 구성해야 할 것이 많다.

- `DataSource`
- 커넥션 풀
- `JdbcTemplate`
- 트랜잭션 관리자
- JPA의 `EntityManagerFactory`
- Hibernate 관련 설정

Spring Boot는 대부분의 애플리케이션이 사용하는 일반적인 구성을 기본값으로 제공한다.

즉, AutoConfiguration의 목적은 다음과 같다.

> 반복적인 인프라 설정은 Spring Boot가 담당하고, 개발자는 애플리케이션의 비즈니스 요구사항에 집중하게 한다.

---

## `@SpringBootApplication`과의 관계

보통 Spring Boot 프로젝트의 메인 클래스에는 다음 annotation이 있다.

```
@SpringBootApplication
public class OrderApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}
```

`@SpringBootApplication`은 크게 다음 세 가지 역할을 합친 annotation이다.

```
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

|Annotation|역할|
|---|---|
|`@SpringBootConfiguration`|이 클래스가 애플리케이션의 핵심 설정 클래스임을 표시|
|`@EnableAutoConfiguration`|AutoConfiguration 활성화|
|`@ComponentScan`|`@Component`, `@Service`, `@Repository`, `@Controller` 등을 검색|

여기서 반드시 구분해야 한다.

- `@ComponentScan`은 **우리가 작성한 컴포넌트**를 찾는다.
- AutoConfiguration은 **Spring Boot 또는 라이브러리가 제공하는 인프라 설정**을 조건에 따라 적용한다.

예를 들어:

```
@Service
public class OrderService {
}
```

`OrderService`가 Bean이 되는 것은 주로 `@ComponentScan` 덕분이다.

반면 `DataSource`, `JdbcTemplate`, `DispatcherServlet` 같은 인프라 Bean이 자동 등록되는 것은 AutoConfiguration 덕분이다.

---

## 전체 동작 흐름

세부 구현을 제외하고 주니어 개발자는 다음 흐름만 이해하면 충분하다.

````
```mermaid
flowchart LR
    A["애플리케이션 실행"] --> B["@SpringBootApplication 처리"]
    B --> C["사용자 컴포넌트와 설정 탐색"]
    C --> D["AutoConfiguration 후보 확인"]
    D --> E["Classpath·Bean·Property 등의 조건 검사"]
    E --> F["조건이 맞는 설정만 적용"]
    F --> G["Bean을 ApplicationContext에 등록"]
    G --> H["의존성 주입 후 애플리케이션 실행"]
```
````

AutoConfiguration은 대략 다음 정보를 보고 판단한다.

1. 어떤 라이브러리가 들어 있는가?
2. 어떤 종류의 애플리케이션인가?
3. 필요한 설정값이 존재하는가?
4. 개발자가 이미 같은 역할의 Bean을 만들었는가?
5. 관련된 다른 Bean이 존재하는가?

---

## 가장 중요한 개념: 조건부 설정

AutoConfiguration의 본질은 `Conditional`, 즉 **조건부 설정**이다.

### `@ConditionalOnClass`

특정 클래스가 Classpath에 있을 때 설정을 적용한다.

```
@ConditionalOnClass(DataSource.class)
```

의미:

> `DataSource` 클래스를 사용할 수 있을 때만 이 설정을 고려한다.

대부분 특정 Starter나 라이브러리를 추가하면 관련 클래스가 Classpath에 들어온다.

---

### `@ConditionalOnMissingBean`

특정 Bean을 개발자가 등록하지 않았을 때 기본 Bean을 등록한다.

```
@Bean
@ConditionalOnMissingBean
public SomeService someService() {
    return new DefaultSomeService();
}
```

의미:

> `SomeService` 타입 Bean이 없으면 기본 구현체를 제공한다.

이 조건이 AutoConfiguration의 아주 중요한 철학인 **Back-off**를 만든다.

---

### `@ConditionalOnBean`

특정 Bean이 존재할 때만 설정을 적용한다.

```
@ConditionalOnBean(DataSource.class)
```

의미:

> `DataSource` Bean이 있는 경우에만 이 설정을 적용한다.

예를 들어 `JdbcTemplate`은 사용할 `DataSource`가 있어야 의미가 있다.

---

### `@ConditionalOnProperty`

설정값에 따라 기능을 켜거나 끈다.

개념적인 예시는 다음과 같다.

```
@ConditionalOnProperty(
    prefix = "app.feature",
    name = "enabled",
    havingValue = "true"
)
```

```
app:
  feature:
    enabled: true
```

의미:

> `app.feature.enabled=true`일 때만 설정을 적용한다.

---

### `@ConditionalOnWebApplication`

현재 애플리케이션이 웹 애플리케이션일 때 적용한다.

웹 환경이 아닌 배치나 CLI 애플리케이션에는 웹 관련 Bean을 만들 필요가 없기 때문이다.

Spring Boot에는 이 밖에도 여러 조건이 있지만, 주니어 개발자라면 위 조건들의 의미를 읽을 수 있으면 충분하다.

---

## Back-off: 개발자가 설정하면 Spring Boot가 물러난다

AutoConfiguration의 가장 중요한 원칙 중 하나다.

> Spring Boot는 기본값을 제공하지만, 개발자가 직접 설정한 부분에서는 물러나는 방식으로 설계되어 있다.

예를 들어 Spring Boot가 기본 `DataSource`를 만들려고 하는 상황에서 개발자가 직접 등록했다고 해보자.

```
@Configuration
public class DatabaseConfig {

    @Bean
    public DataSource dataSource() {
        // 직접 구성
        return createCustomDataSource();
    }
}
```

AutoConfiguration에 `@ConditionalOnMissingBean(DataSource.class)` 조건이 있다면 Spring Boot는 기본 `DataSource`를 만들지 않는다.

하지만 다음 사항은 주의해야 한다.

- 개발자 Bean이 있다고 해서 모든 AutoConfiguration이 중단되는 것은 아니다.
- **어떤 AutoConfiguration이 물러나는지는 각각의 조건에 달려 있다.**
- 직접 만든 `DataSource`를 사용해 JPA나 `JdbcTemplate` 관련 AutoConfiguration이 계속 적용될 수도 있다.

따라서 “내가 Bean 하나를 만들면 Boot 설정 전체가 꺼진다”는 이해는 잘못된 것이다.

---

## Starter와 AutoConfiguration은 같은 것이 아니다

둘은 같이 사용되는 경우가 많지만 역할이 다르다.

### Starter

필요한 의존성을 편하게 모아 놓은 의존성 묶음이다.

예:

```
implementation("org.springframework.boot:spring-boot-starter-data-jpa")
```

일반적으로 JPA를 사용하는 데 필요한 Spring Data JPA, ORM 구현체 등의 의존성이 함께 들어온다. Starter는 편리한 의존성 descriptor라는 것이 공식 정의다.

### AutoConfiguration

Starter나 다른 의존성으로 들어온 라이브러리를 발견하여 실제 Bean과 기본 설정을 구성하는 기능이다.

관계를 정리하면 다음과 같다.

```
Starter 추가
    ↓
관련 라이브러리가 Classpath에 들어옴
    ↓
AutoConfiguration의 조건이 충족될 가능성이 생김
    ↓
조건이 모두 맞으면 필요한 Bean이 등록됨
```

Starter를 추가했다고 반드시 모든 관련 Bean이 만들어지는 것은 아니다. 설정값이나 다른 Bean 등 추가 조건이 필요할 수 있다.

---

## 설정값은 AutoConfiguration을 조절하는 첫 번째 방법이다

Spring Boot의 기본 동작을 바꾸고 싶다고 바로 설정 클래스를 작성할 필요는 없다. 먼저 `application.yml` 또는 `application.properties`에서 제공되는 설정값을 찾아야 한다.

```
server:
  port: 8081

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/order
    username: order_app
    password: ${DB_PASSWORD}

  jpa:
    hibernate:
      ddl-auto: validate
```

AutoConfiguration은 최종적으로 결정된 환경 설정값을 사용한다.

설정값은 다음 경로 등에서 들어올 수 있다.

- `application.yml`
- `application-{profile}.yml`
- 환경변수
- JVM 시스템 속성
- 명령행 인자

```
java -jar app.jar --server.port=9090
```

일반적으로 프로필별 설정은 공통 설정을 덮어쓰고, 명령행 설정은 파일 설정보다 우선한다.

커스터마이징 우선순위는 보통 다음처럼 생각하면 된다.

1. `application.yml` 설정값 사용
2. Spring Boot가 제공하는 `Customizer`나 `Configurer` 사용
3. 필요한 Bean을 직접 등록
4. 특정 AutoConfiguration 제외 후 전체 수동 설정

---

## 패키지 위치

메인 클래스는 일반적으로 프로젝트 최상위 패키지에 둔다.

```
com.example.order
├── OrderApplication.java
├── order
│   ├── OrderController.java
│   └── OrderService.java
└── payment
    └── PaymentService.java
```

```
package com.example.order;

@SpringBootApplication
public class OrderApplication {
}
```

`@ComponentScan`은 기본적으로 메인 클래스가 속한 패키지와 그 하위 패키지를 검색한다.

따라서 다음 구조라면 `PaymentService`를 찾지 못할 수 있다.

```
com.example.order.OrderApplication
com.example.payment.PaymentService
```

JPA Entity와 Repository를 검색하는 기본 패키지에도 메인 클래스 위치가 영향을 줄 수 있다. 공식 문서도 메인 클래스를 다른 애플리케이션 클래스보다 위쪽의 루트 패키지에 두는 것을 권장한다.

---

## AutoConfiguration 문제를 확인하는 방법

AutoConfiguration과 관련된 문제가 발생하면 추측만 하지 말고 **조건 평가 결과**를 봐야 한다.

### `--debug` 옵션

```
java -jar app.jar --debug
```

Gradle:

```
./gradlew bootRun --args='--debug'
```

Maven:

```
./mvnw spring-boot:run -Dspring-boot.run.arguments=--debug
```

실행 로그에 Condition Evaluation Report가 출력된다.

주로 다음 내용을 볼 수 있다.

- `Positive matches`: 조건을 통과한 AutoConfiguration
- `Negative matches`: 조건을 통과하지 못한 AutoConfiguration
- `Exclusions`: 명시적으로 제외한 AutoConfiguration
- 조건이 통과하거나 실패한 이유

`Negative matches`가 많아도 오류는 아니다. 현재 애플리케이션에 필요하지 않은 설정이 제외된 정상적인 결과가 대부분이다.

예를 들어 다음처럼 해석할 수 있다.

```
DataSourceAutoConfiguration:
    Did not match:
        DataSource class was not found
```

의미:

> JDBC 관련 클래스가 Classpath에 없어서 DataSource 자동 설정을 적용하지 않았다.

```
SomeAutoConfiguration:
    Did not match:
        SomeService bean was already found
```

의미:

> 개발자가 이미 관련 Bean을 등록했기 때문에 기본 Bean을 등록하지 않았다.

Spring Boot도 AutoConfiguration 적용 여부를 확인할 때 우선 `--debug`를 사용하라고 안내한다.

---

### Actuator의 `conditions` endpoint

Actuator를 사용하는 프로젝트에서는 `/actuator/conditions`를 통해 조건 평가 결과를 확인할 수 있다.

```
management:
  endpoints:
    web:
      exposure:
        include: health,conditions
```

```
GET /actuator/conditions
```

다만 기본적으로 HTTP에는 `health`만 노출되며, `conditions`에는 내부 구성 정보가 포함될 수 있으므로 운영 환경에서는 인증·인가나 네트워크 접근 통제가 필요하다.

---

## AutoConfiguration을 제외하는 방법

특정 AutoConfiguration이 정말 필요 없을 때는 제외할 수 있다.

```
@SpringBootApplication(
    exclude = DataSourceAutoConfiguration.class
)
public class OrderApplication {
}
```

설정 파일로도 제외할 수 있다.

```
spring:
  autoconfigure:
    exclude:
      - 전체.패키지명.DataSourceAutoConfiguration
```

하지만 제외는 보통 마지막 선택이다.

DB 오류가 발생했다는 이유만으로 `DataSourceAutoConfiguration`을 바로 제외하면 안 된다. 실제 원인이 다음 중 하나일 수 있기 때문이다.

- DB 의존성을 실수로 추가함
- JDBC Driver가 없음
- URL 설정이 잘못됨
- 잘못된 프로필이 활성화됨
- 환경변수가 전달되지 않음

먼저 “왜 이 AutoConfiguration이 활성화되었는가?”를 확인하고, 그 기능 자체가 필요 없을 때 제외해야 한다.

---

## 주니어 개발자가 자주 하는 실수

### `@EnableWebMvc`를 무심코 추가한다

Spring Boot의 MVC 기본 설정을 유지하면서 인터셉터 등을 추가하려면 보통 `WebMvcConfigurer`를 사용한다.

```
@Configuration
public class WebConfig implements WebMvcConfigurer {
}
```

이때 `@EnableWebMvc`를 붙이면 Spring Boot의 MVC AutoConfiguration에서 벗어나 직접 제어하는 방향으로 바뀔 수 있다.

```
@Configuration
@EnableWebMvc // Boot 기본 MVC 설정을 사용하려는 경우 주의
public class WebConfig {
}
```

단순 커스터마이징이라면 보통 `@EnableWebMvc` 없이 `WebMvcConfigurer`를 사용한다.

---

### Starter를 추가하면 기능이 무조건 완성된다고 생각한다

Starter는 의존성을 넣어 준다. AutoConfiguration은 조건을 검사한다.

DB Starter를 추가했지만 DB URL이나 Driver가 없다면 애플리케이션 실행이 실패할 수 있다.

---

### 모든 Bean이 AutoConfiguration으로 생긴다고 생각한다

다음은 컴포넌트 스캔으로 등록되는 사용자 Bean이다.

```
@Service
public class PaymentService {
}
```

다음은 AutoConfiguration으로 만들어질 가능성이 있는 인프라 Bean이다.

- `DataSource`
- `JdbcTemplate`
- `DispatcherServlet`
- JSON Mapper
- 트랜잭션 관리자

---

### 직접 만든 Bean이 항상 Boot 기본 설정을 대체한다고 생각한다

AutoConfiguration마다 조건이 다르다.

- 타입 기준으로 확인할 수 있음
- Bean 이름 기준으로 확인할 수 있음
- 특정 Bean의 존재 여부를 확인할 수 있음
- 설정값까지 함께 확인할 수 있음

따라서 “내 Bean이 있으니 당연히 기본 Bean은 없어지겠지”라고 추측하면 안 된다. Condition Evaluation Report를 확인해야 한다.

---

### 테스트에서 Bean이 없다고 운영 설정까지 의심한다

`@SpringBootTest`는 전체 애플리케이션에 가까운 Context를 구성한다.

반면 다음과 같은 테스트 slice는 필요한 영역만 제한적으로 구성한다.

- `@WebMvcTest`: MVC 관련 영역
- `@DataJpaTest`: JPA 관련 영역
- `@JsonTest`: JSON 직렬화 관련 영역

따라서 `@WebMvcTest`에서 Service Bean이 없다고 해서 실제 애플리케이션의 AutoConfiguration이나 컴포넌트 스캔이 잘못된 것은 아닐 수 있다. 테스트 slice가 의도적으로 일부만 로딩한 것이다.