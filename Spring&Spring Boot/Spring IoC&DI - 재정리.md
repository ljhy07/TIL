# Spring IoC & DI

> **IoC는 객체를 만들고 연결하고 관리하는 주도권을 개발자 코드가 아니라 Spring 컨테이너에 맡기는 원칙이다.**

> **DI는 객체가 필요로 하는 의존성을 객체 외부에서 넣어 주는 방식이며, Spring이 IoC를 구현하는 대표적인 방법이다.**

Spring 공식 문서도 DI를 IoC의 특수한 형태로 설명한다. 즉, 둘을 완전히 같은 말로 보기보다는 **IoC가 더 큰 개념이고 DI가 이를 실현하는 방법**이라고 이해하는 것이 정확한다.

---

## 의존성이란 무엇인가

어떤 클래스가 작업을 수행하기 위해 다른 객체를 필요로 한다면, 그 객체를 해당 클래스의 **의존성Dependency**이라고 한다.

```
public class OrderService {

    private final PaymentGateway paymentGateway;

    public OrderService() {
        this.paymentGateway = new KakaoPayGateway();
    }

    public void order() {
        paymentGateway.pay();
    }
}
```

여기서 `OrderService`는 결제하기 위해 `PaymentGateway`가 필요한다.

따라서 다음 관계가 성립한다.

```
OrderService → PaymentGateway에 의존한다
```

그런데 위 코드에서는 `OrderService`가 직접 `KakaoPayGateway`를 생성한다.

```
this.paymentGateway = new KakaoPayGateway();
```

이 경우 `OrderService`가 다음 사항을 모두 결정한다.

- 어떤 결제 구현체를 사용할지
- 객체를 언제 생성할지
- 객체를 어떻게 생성할지
- 해당 객체를 얼마나 오래 사용할지

이처럼 객체가 자신의 의존성을 직접 생성하면 두 클래스 사이의 결합이 강해진다.

---

## 직접 객체를 생성하면 무엇이 불편한가

### 구현체를 변경하기 어렵다

카카오페이에서 네이버페이로 변경하려면 `OrderService` 코드를 직접 수정해야 한다.

```
this.paymentGateway = new NaverPayGateway();
```

결제 방식이라는 외부 정책이 바뀌었는데 주문 서비스 코드까지 수정된다.

### 단위 테스트가 어렵다

테스트에서는 실제 결제 서버를 호출하지 않고 가짜 객체를 사용하고 싶을 수 있다.

하지만 `OrderService` 내부에서 직접 `KakaoPayGateway`를 만들면 테스트 코드에서 이를 바꾸기 어렵다.

### 객체 생성 코드가 여기저기 퍼진다

여러 클래스에서 다음 코드가 반복될 수 있다.

```
new KakaoPayGateway(...);
```

생성자 인자나 설정 방법이 바뀌면 여러 코드를 함께 수정해야 한다.

---

## DI를 적용한 코드

먼저 결제 기능을 인터페이스로 분리한다.

```
public interface PaymentGateway {
    void pay();
}
```

구현체를 작성한다.

```
@Component
public class KakaoPayGateway implements PaymentGateway {

    @Override
    public void pay() {
        System.out.println("카카오페이 결제");
    }
}
```

`OrderService`는 구현체를 직접 생성하지 않고 생성자를 통해 전달받는다.

```
@Service
public class OrderService {

    private final PaymentGateway paymentGateway;

    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public void order() {
        paymentGateway.pay();
    }
}
```

이제 `OrderService`는 다음 사실만 알고 있다.

> “나는 `PaymentGateway`가 필요하다.”

반면 다음 사항은 몰라도 된다.

- 실제 구현체가 `KakaoPayGateway`인지
- 누가 구현체를 생성했는지
- 구현체가 어떤 설정값을 필요로 하는지

Spring이 애플리케이션을 시작하면서 대략 다음 작업을 수행한다.

1. `@Component`, `@Service` 등이 붙은 클래스를 찾는다.
2. 해당 클래스의 객체를 생성한다.
3. `OrderService` 생성자를 확인한다.
4. `PaymentGateway` 타입의 Bean을 찾는다.
5. 찾은 객체를 `OrderService` 생성자에 전달한다.
6. 생성된 객체들을 컨테이너 안에서 관리한다.

객체 생성과 연결의 주도권이 `OrderService`에서 Spring으로 넘어갔으므로 **제어가 역전되었다IoC**고 표현한다.

---

## Bean과 IoC 컨테이너

### Bean

**Spring IoC 컨테이너가 생성하고 관리하는 객체**를 Bean이라고 한다.

```
@Service
public class OrderService {
}
```

위 클래스가 컴포넌트 스캔을 통해 등록되면 `OrderService` 객체는 Bean이다.

하지만 다음과 같이 직접 생성한 객체는 일반적으로 Spring Bean이 아니다.

```
OrderService orderService = new OrderService(...);
```

모든 Java 객체가 Bean인 것은 아니다.

```
Java 객체
├── Spring이 관리하는 객체 → Bean
└── 개발자가 직접 생성하고 관리하는 객체 → 일반 객체
```

### IoC 컨테이너

Bean을 생성하고, 의존성을 연결하고, 생명주기를 관리하는 공간이다.

Spring에서는 주로 `ApplicationContext`가 IoC 컨테이너 역할을 한다. `BeanFactory`는 기본 기능을 제공하고, `ApplicationContext`가 이를 확장하지만 **실제 애플리케이션에서 주로 사용하는 컨테이너가 `ApplicationContext`**라는 정도면 충분하다.

Spring Boot에서는 다음 코드가 실행될 때 컨테이너가 만들어진다.

```
@SpringBootApplication
public class OrderApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}
```

---

## Bean을 등록하는 대표적인 두 가지 방법

### 방법 1: 컴포넌트 스캔

클래스에 다음과 같은 애너테이션을 붙인다.

```
@Component
public class PasswordEncoder {
}
```

```
@Service
public class OrderService {
}
```

```
@Repository
public class OrderRepository {
}
```

```
@Controller
public class OrderController {
}
```

이 애너테이션들은 모두 컴포넌트 스캔 대상이다.

|애너테이션|일반적인 용도|
|---|---|
|`@Component`|일반적인 Spring 컴포넌트|
|`@Service`|비즈니스 로직|
|`@Repository`|DB 접근, 저장소 계층|
|`@Controller`|Spring MVC 컨트롤러|
|`@RestController`|REST API 컨트롤러|

`@Service`, `@Repository`, `@Controller`는 모두 `@Component`를 기반으로 만들어진 특화 애너테이션이다. 역할에 맞는 애너테이션을 사용하는 것이 좋다. 특히 `@Repository`는 데이터 접근 예외 변환 등의 추가 의미도 가질 수 있다.

### 컴포넌트 스캔 범위 주의

`@SpringBootApplication`은 기본적으로 해당 클래스가 위치한 패키지와 하위 패키지를 스캔한다.

```
com.example.order
├── OrderApplication.java      ← @SpringBootApplication
├── controller
├── service
└── repository
```

따라서 메인 클래스를 프로젝트의 최상위 패키지에 두는 것이 일반적이다.

다음처럼 스캔 대상이 상위나 형제 패키지에 있으면 Bean으로 등록되지 않을 수 있다.

```
com.example.application.OrderApplication
com.example.service.OrderService
```

`OrderApplication`이 `com.example.application`에 있으면 기본 설정으로는 `com.example.service`를 스캔하지 못한다. Spring Boot 공식 문서도 메인 클래스를 루트 패키지에 두는 구성을 권장한다.

---

### 방법 2: `@Configuration`과 `@Bean`

외부 라이브러리의 클래스처럼 직접 `@Component`를 붙일 수 없거나, 객체 생성 과정을 명시적으로 설정하고 싶을 때 사용한다.

```
@Configuration
public class PaymentConfig {

    @Bean
    public PaymentClient paymentClient() {
        return new PaymentClient("https://payment.example.com");
    }
}
```

`paymentClient()`가 반환한 객체가 Spring Bean으로 등록된다.

`@Bean` 메서드도 다른 Bean을 매개변수로 주입받을 수 있다.

```
@Configuration
public class PaymentConfig {

    @Bean
    public PaymentClient paymentClient(PaymentProperties properties) {
        return new PaymentClient(properties.getBaseUrl());
    }
}
```

다음의 기준으로 선택하면 좋다.

- 직접 작성한 서비스, 컨트롤러, 저장소 → `@Service`, `@Controller`, `@Repository`
- 외부 라이브러리 객체 또는 생성 과정을 직접 제어해야 하는 객체 → `@Bean`

---

## DI 방법과 권장 순서

Spring은 생성자, Setter, 필드 등에 의존성을 주입할 수 있다.

### 1순위: 생성자 주입

```
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;

    public OrderService(
            OrderRepository orderRepository,
            PaymentGateway paymentGateway
    ) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
    }
}
```

생성자 주입이 권장되는 이유는 다음과 같다.

- 필수 의존성이 누락된 객체가 생성되는 것을 막을 수 있다.
- 필드를 `final`로 선언할 수 있다.
- 클래스가 무엇을 필요로 하는지 생성자만 보고 알 수 있다.
- Spring 없이 순수한 Java 단위 테스트를 작성하기 쉽다.
- 순환 의존성을 애플리케이션 시작 시점에 발견하기 쉽다.

Spring 공식 문서도 필수 의존성에는 생성자 주입을 권장한다.

생성자가 하나뿐이라면 `@Autowired`를 생략할 수 있다.

```
@Service
public class OrderService {

    private final OrderRepository orderRepository;

    // @Autowired가 없어도 주입됨
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}
```

Lombok을 사용한다면 다음과 같이 줄일 수도 있다.

```
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
}
```

`@RequiredArgsConstructor`가 `final` 필드를 받는 생성자를 만들어 줄 뿐, DI 원리가 바뀌는 것은 아니다.

---

### Setter 주입

```
@Service
public class OrderService {

    private DiscountPolicy discountPolicy;

    @Autowired
    public void setDiscountPolicy(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
}
```

Setter 주입은 의존성이 선택 사항이고, 의존성이 없어도 객체가 정상적으로 동작할 수 있을 때 고려할 수 있다.

그러나 대부분의 서비스 의존성은 필수이므로 생성자 주입이 더 적합하다.

---

### 필드 주입

```
@Service
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;
}
```

작동은 하지만 일반적인 애플리케이션 코드에서는 권장하지 않는다.

- 의존성이 숨겨진다.
- `final`을 사용할 수 없다.
- Spring 없이 객체를 생성하면 필드가 `null`이다.
- 단위 테스트에서 의존성을 전달하기 어렵다.
- 의존성이 너무 많은 클래스라는 사실을 알아보기 어렵다.

특히 다음 코드는 `orderRepository`가 주입되지 않는다.

```
OrderService orderService = new OrderService();
```

이 객체는 Spring이 생성한 객체가 아니기 때문입니다.

---

## `@Autowired`는 기본적으로 타입으로 찾는다

다음 생성자가 있다고 가정한다.

```
public OrderService(PaymentGateway paymentGateway) {
    this.paymentGateway = paymentGateway;
}
```

Spring은 `PaymentGateway` 타입으로 등록된 Bean을 찾는다.

### 후보가 하나인 경우

```
@Component
public class KakaoPayGateway implements PaymentGateway {
}
```

`KakaoPayGateway`가 주입된다.

### 후보가 없는 경우

Spring은 `OrderService`를 생성할 수 없으므로 일반적으로 애플리케이션 시작에 실패한다.

주로 다음을 확인해야 한다.

- 구현체에 `@Component`가 빠졌는가
- 컴포넌트 스캔 범위 밖에 있는가
- `@Bean` 등록이 누락됐는가
- 의존성 라이브러리가 빠졌는가

### 후보가 여러 개인 경우

```
@Component
public class KakaoPayGateway implements PaymentGateway {
}

@Component
public class NaverPayGateway implements PaymentGateway {
}
```

Spring 입장에서는 무엇을 주입해야 할지 결정할 수 없다. 이때 `NoUniqueBeanDefinitionException`과 같은 오류가 발생할 수 있다.

---

## 구현체가 여러 개일 때: `@Primary`와 `@Qualifier`

### `@Primary`

기본으로 사용할 구현체를 지정한다.

```
@Component
@Primary
public class KakaoPayGateway implements PaymentGateway {
}
```

```
@Component
public class NaverPayGateway implements PaymentGateway {
}
```

이 경우 특별한 지정이 없으면 `KakaoPayGateway`가 주입된다.

### `@Qualifier`

특정 구현체를 명시적으로 선택한다.

```
@Component("kakaoPayGateway")
public class KakaoPayGateway implements PaymentGateway {
}
```

```
@Component("naverPayGateway")
public class NaverPayGateway implements PaymentGateway {
}
```

```
@Service
public class OrderService {

    private final PaymentGateway paymentGateway;

    public OrderService(
            @Qualifier("naverPayGateway")
            PaymentGateway paymentGateway
    ) {
        this.paymentGateway = paymentGateway;
    }
}
```

기준은 다음처럼 이해하면 된다.

- 애플리케이션의 일반적인 기본값 → `@Primary`
- 특정 위치에서 명확하게 구현체 선택 → `@Qualifier`

Spring은 기본적으로 타입 후보를 먼저 찾고 `@Qualifier`로 그 후보를 좁힙니다.

---

## 인터페이스가 있어야만 DI인가

아니다. 구체 클래스도 주입할 수 있다.

```
@Service
public class OrderValidator {
}
```

```
@Service
public class OrderService {

    private final OrderValidator orderValidator;

    public OrderService(OrderValidator orderValidator) {
        this.orderValidator = orderValidator;
    }
}
```

이것도 생성자 DI이다.

인터페이스의 장점은 구현 교체가 필요한 경계를 분리할 수 있다는 점이다.

```
PaymentGateway
├── KakaoPayGateway
├── NaverPayGateway
└── FakePaymentGateway
```

하지만 모든 클래스마다 무조건 인터페이스를 만드는 것도 좋은 설계는 아니다.

인터페이스는 주로 다음 상황에서 유용하다.

- 구현체가 실제로 여러 개 존재한다.
- 외부 시스템과의 연결을 분리한다.
- 테스트용 대체 구현이 필요하다.
- 구현체가 변경될 가능성이 의미 있게 존재한다.

---

## DI가 테스트를 쉽게 만드는 이유

생성자 주입을 사용하면 Spring을 실행하지 않고도 테스트할 수 있다.

```
class FakePaymentGateway implements PaymentGateway {

    private boolean paid;

    @Override
    public void pay() {
        paid = true;
    }

    public boolean isPaid() {
        return paid;
    }
}
```

```
@Test
void 주문하면_결제한다() {
    FakePaymentGateway fakeGateway = new FakePaymentGateway();
    OrderService orderService = new OrderService(fakeGateway);

    orderService.order();

    assertTrue(fakeGateway.isPaid());
}
```

테스트가 직접 원하는 구현체를 전달할 수 있다.

```
new OrderService(fakeGateway);
```

이것이 DI의 중요한 장점이다. DI는 단순히 애너테이션을 사용하기 위한 기술이 아니라 **객체를 조립하는 방법을 객체의 비즈니스 로직과 분리하는 설계 방식**이다.

---

## Bean Scope와 singleton

Spring Bean의 기본 Scope는 `singleton`이다.

동일한 Spring 컨테이너 안에서 하나의 Bean 정의에 대해 기본적으로 하나의 객체를 만들어 공유한다.

```
@Service
public class OrderService {
}
```

여러 컨트롤러가 `OrderService`를 주입받더라도 일반적으로 동일한 인스턴스를 공유한다.

### Spring singleton과 일반 Singleton Pattern의 차이

Spring singleton은 다음 의미한다.

> Bean 하나당 Spring 컨테이너 내부에 객체 인스턴스 하나

반드시 JVM 전체에 객체가 하나라는 뜻은 아니다. `ApplicationContext`가 여러 개라면 컨텍스트마다 인스턴스가 존재할 수 있다.

### singleton Bean에 요청별 상태를 저장하면 안 된다

다음 코드는 위험하다.

```
@Service
public class OrderService {

    private Long currentUserId;

    public void order(Long userId) {
        this.currentUserId = userId;
    }
}
```

하나의 `OrderService` 인스턴스를 여러 요청이 동시에 사용하기 때문에 사용자 값이 섞일 수 있다.

요청마다 달라지는 값은 필드가 아니라 메서드 인자나 지역 변수로 다루는 것이 기본이다.

```
@Service
public class OrderService {

    public void order(Long userId) {
        Long currentUserId = userId;
        // 처리
    }
}
```

Scope에 대해 가장 먼저 기억할 것은 다음이다.

> **일반적인 `@Service`, `@Repository`, `@Controller`는 공유되는 singleton Bean이므로 가능하면 상태를 가지지 않게 설계한다.**

---

## Bean의 기본 생명주기

복잡한 내부 구현을 제외하면 다음 순서로 이해하면 된다.

```
설정과 컴포넌트 탐색
    ↓
Bean 객체 생성
    ↓
의존성 주입
    ↓
초기화 콜백
    ↓
애플리케이션에서 사용
    ↓
컨테이너 종료
    ↓
종료 콜백
```

초기화와 종료 시점에 작업이 필요하다면 `@PostConstruct`, `@PreDestroy`를 사용할 수 있다.

```
@Component
public class ExternalClient {

    @PostConstruct
    public void init() {
        System.out.println("초기화");
    }

    @PreDestroy
    public void close() {
        System.out.println("자원 정리");
    }
}
```

현대 Spring에서는 다음 패키지를 사용한다.

```
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
```

- `@PostConstruct`는 의존성 주입이 끝난 뒤 호출된다.
- `@PreDestroy`는 컨테이너가 Bean을 제거하기 전에 호출된다.

---

## 순환 의존성

다음과 같은 관계를 순환 의존성이라고 한다.

```
OrderService → PaymentService
PaymentService → OrderService
```

코드로 보면 다음과 같다.

```
@Service
public class OrderService {

    public OrderService(PaymentService paymentService) {
    }
}
```

```
@Service
public class PaymentService {

    public PaymentService(OrderService orderService) {
    }
}
```

Spring은 `OrderService`를 만들기 위해 `PaymentService`가 필요하고, `PaymentService`를 만들기 위해 다시 `OrderService`가 필요한 상태가 된다.

생성자 기반 순환 의존성은 정상적으로 객체를 완성할 수 없으므로 애플리케이션 시작에 실패한다.

해결 방법은 보통 애너테이션을 추가하는 것이 아니라 설계를 다시 살펴보는 것이다.

- 한쪽 의존 방향을 제거한다.
- 공통 책임을 별도의 클래스로 분리한다.
- 두 서비스의 역할이 과도하게 얽혀 있지 않은지 확인한다.

`@Lazy`나 필드 주입으로 오류를 억지로 피하기 전에 구조적 문제를 먼저 확인해야 한다.

---

## `new`를 사용하면 안 된다는 뜻은 아니다

IoC와 DI를 배운 뒤 흔히 하는 오해이다.

> “Spring에서는 `new`를 사용하면 안 된다.”

그렇지 않습니다.

다음과 같은 값 객체나 DTO, Entity는 필요할 때 직접 생성하는 경우가 많다.

```
Order order = new Order(...);
Money money = new Money(10_000);
OrderResponse response = new OrderResponse(...);
```

주로 Bean으로 관리하는 객체는 다음과 같다.

- Controller
- Service
- Repository
- 외부 시스템 Client
- 설정 및 인프라 객체
- 여러 곳에서 공유되는 정책과 어댑터

반면 요청 데이터, 응답 DTO, Entity, 단순 값 객체까지 전부 Bean으로 만들 필요는 없다.

핵심 기준:
> **다른 애플리케이션 컴포넌트와 연결되고 Spring이 생명주기나 설정을 관리할 필요가 있는 객체인가?**

---

## 자주 발생하는 실수

### 클래스에 애너테이션만 붙이고 직접 `new`를 사용한다

```
@Service
public class OrderService {
}
```

```
OrderService orderService = new OrderService();
```

직접 생성한 객체는 Spring Bean이 아니므로 Spring의 DI나 생명주기 관리를 받지 못한다.

### Bean이 컴포넌트 스캔 범위 밖에 있다

메인 애플리케이션 클래스의 패키지 구조를 확인해야 한다.

### 같은 인터페이스 구현체가 여러 개인데 구분하지 않는다

`@Primary` 또는 `@Qualifier`가 필요하다.

### singleton Bean에 사용자별 값을 필드로 저장한다

동시 요청에서 값이 섞일 수 있다.

### 생성자 인자가 지나치게 많다

```
public OrderService(
        A a,
        B b,
        C c,
        D d,
        E e,
        F f,
        G g
) {
}
```

생성자 주입이 문제라기보다는 클래스가 너무 많은 책임을 갖고 있다는 신호일 가능성이 크다.

### 비즈니스 코드에서 `ApplicationContext.getBean()`을 반복 사용한다

```
PaymentGateway gateway =
        applicationContext.getBean(PaymentGateway.class);
```

이 방식은 클래스의 의존성을 숨기고, 클래스가 Spring 컨테이너를 직접 조회하게 만든다. 일반적인 서비스 코드에서는 필요한 Bean을 생성자로 전달받는 것이 좋다.