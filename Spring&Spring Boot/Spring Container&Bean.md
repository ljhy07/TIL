## Spring IoC 컨테이너

Spring에서 객체를 생성하고 관리하는 중심 구성 요소가 IoC 컨테이너이다.

핵심 인터페이스의 관계는 다음과 같다.

```text
BeanFactory
└── ApplicationContext
```

### BeanFactory

`BeanFactory`는 Spring 컨테이너의 가장 기본적인 기능을 정의한다.

- Bean 등록 정보 관리
- Bean 생성
- 의존관계 주입
- Bean 조회
- Bean 스코프 관리

```java
PaymentService paymentService =
        beanFactory.getBean(PaymentService.class);
```

### ApplicationContext

실제 Spring 애플리케이션에서는 대부분 `ApplicationContext`를 사용한다. `BeanFactory`의 기능에 다음과 같은 애플리케이션 기능이 추가된다.

- 환경 변수와 프로퍼티 처리
- 리소스 로딩
- 이벤트 발행과 구독
- 국제화
- 애노테이션 기반 설정
- Bean 후처리기 자동 등록
- AOP 및 프록시 인프라 통합

Spring Boot 애플리케이션을 실행하는 다음 코드도 내부적으로 `ApplicationContext`를 생성한다.

```java
SpringApplication.run(ShopApplication.class, args);
```

## Spring Bean

Spring Bean은 Spring IoC 컨테이너가 생성하고 구성하며 생명주기를 관리하는 객체이다.

```java
PaymentService service = new KakaoPaymentService();
```

위 객체는 일반 Java 객체이다. 동일한 클래스라도 Spring이 관리하면 Bean이 된다.

```java
@Service
public class KakaoPaymentService
        implements PaymentService {
}
```

Bean은 컨테이너의 관리 대상이므로 다음과 같은 Spring 기능과 결합될 수 있다.

- 의존관계 주입
- 생명주기 콜백
- 스코프 관리
- AOP 프록시
- 트랜잭션
- 캐시
- 비동기 실행

다만 Bean으로 등록되었다는 사실만으로 모든 기능이 자동 적용되는 것은 아니다. 해당 기능에 필요한 설정과 후처리기가 활성화되어 있어야 한다.

## 컴포넌트 스캔으로 등록하기

클래스에 `@Component`를 붙이면 컴포넌트 스캔이 클래스를 발견하여 Bean으로 등록할 수 있다.

```java
@Component
public class EmailSender {
}
```

역할을 나타내는 특화 애노테이션도 있다.

```java
@Controller
@RestController
@Service
@Repository
@Configuration
```

이 애노테이션들은 메타 애노테이션으로 `@Component`를 포함하므로 컴포넌트 스캔의 대상이 된다.

```java
@Service
public class OrderService {
}
```

```java
@Repository
public class OrderRepository {
}
```

Spring Boot에서는 일반적으로 `@SpringBootApplication`이 선언된 클래스의 패키지부터 하위 패키지를 탐색한다.

```java
@SpringBootApplication
public class ShopApplication {

    public static void main(String[] args) {
        SpringApplication.run(
                ShopApplication.class, args);
    }
}
```

```text
com.example.shop
├── ShopApplication
├── order
├── payment
└── member
```

스캔 대상 클래스가 시작 클래스의 상위 패키지나 전혀 다른 패키지에 있으면 자동으로 발견되지 않을 수 있다.

## Java 설정으로 등록하기

`@Configuration`과 `@Bean`을 사용하면 객체 생성 방법을 명시할 수 있다.

```java
@Configuration
public class AppConfig {

    @Bean
    public PaymentService paymentService() {
        return new KakaoPaymentService();
    }

    @Bean
    public OrderService orderService(
            PaymentService paymentService) {

        return new OrderService(paymentService);
    }
}
```

`@Bean` 메서드의 반환 객체가 컨테이너에 등록된다. 메서드의 매개변수는 Spring이 다른 Bean을 찾아 주입한다.

Java 설정은 다음과 같은 상황에서 특히 유용하다.

- 소스 코드를 수정할 수 없는 외부 라이브러리 객체 등록
- 복잡한 생성 과정 표현
- 설정값을 이용한 객체 생성
- 환경이나 조건에 따른 구현체 등록

```java
@Configuration
public class ClientConfig {

    @Bean
    public ExternalApiClient externalApiClient(
            ApiProperties properties) {

        return new ExternalApiClient(
                properties.baseUrl(),
                properties.apiKey()
        );
    }
}
```

## Bean 이름

컴포넌트 방식에서 기본 Bean 이름은 일반적으로 클래스 이름의 첫 글자를 소문자로 바꾼 형태이다.

```java
@Component
public class KakaoPaymentService {
}
```

기본 이름은 보통 다음과 같다.

```text
kakaoPaymentService
```

이름을 직접 지정할 수도 있다.

```java
@Component("kakaoPayment")
public class KakaoPaymentService {
}
```

```java
@Bean("kakaoPayment")
public PaymentService paymentService() {
    return new KakaoPaymentService();
}
```

Bean 이름은 이름 기반 조회, `Map<String, T>` 주입, 한정자 처리 등에서 사용될 수 있다.

## XML 설정

XML에서도 Bean과 의존관계를 정의할 수 있다.

```xml
<bean id="paymentService"
      class="com.example.KakaoPaymentService"/>

<bean id="orderService"
      class="com.example.OrderService">
    <constructor-arg ref="paymentService"/>
</bean>
```

Spring Boot 애플리케이션에서는 컴포넌트 스캔과 Java 설정이 일반적이지만, XML 기반 레거시 시스템에서는 같은 컨테이너 원리를 XML로 표현한 코드를 볼 수 있다.

## 조건에 따라 Bean 등록하기

환경에 따라 서로 다른 구현체를 등록할 수 있다.

```java
@Configuration
@Profile("local")
public class LocalPaymentConfig {

    @Bean
    public PaymentService paymentService() {
        return new FakePaymentService();
    }
}
```

```java
@Configuration
@Profile("prod")
public class ProductionPaymentConfig {

    @Bean
    public PaymentService paymentService() {
        return new KakaoPaymentService();
    }
}
```

Spring Boot에서는 프로퍼티 조건도 사용할 수 있다.

```java
@Bean
@ConditionalOnProperty(
    name = "payment.provider",
    havingValue = "kakao"
)
public PaymentService kakaoPaymentService() {
    return new KakaoPaymentService();
}
```

이 경우 사용하는 쪽은 동일한 `PaymentService` 타입을 주입받지만, 실제 구현체는 실행 환경의 설정에 따라 달라질 수 있다.

