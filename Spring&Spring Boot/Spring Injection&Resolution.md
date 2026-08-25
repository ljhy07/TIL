## 생성자 주입

생성자의 매개변수를 통해 의존 객체를 전달한다.

```java
@Service
public class OrderService {

    private final PaymentService paymentService;
    private final OrderRepository orderRepository;

    public OrderService(
            PaymentService paymentService,
            OrderRepository orderRepository) {

        this.paymentService = paymentService;
        this.orderRepository = orderRepository;
    }
}
```

생성자가 하나라면 일반적으로 `@Autowired`를 생략할 수 있다.

```java
@Service
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

생성자 주입은 객체 생성 시점에 필수 의존성을 모두 전달한다. 따라서 의존성이 없는 불완전한 객체가 만들어지는 것을 막을 수 있고 필드를 `final`로 유지할 수 있다.

## Setter 주입

Setter 메서드에 의존 객체를 전달한다.

```java
@Service
public class OrderService {

    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(
            PaymentService paymentService) {

        this.paymentService = paymentService;
    }
}
```

객체 생성 이후에 의존관계가 설정된다. 선택적으로 설정하거나 나중에 변경해야 하는 프로퍼티에 사용할 수 있지만, 필수 의존성에 사용하면 주입 전의 불완전한 객체가 존재할 수 있다.

## 필드 주입

```java
@Service
public class OrderService {

    @Autowired
    private PaymentService paymentService;
}
```

Spring은 리플렉션과 후처리기를 이용하여 필드에 값을 넣는다. 코드는 짧지만 생성자만 보고 객체가 필요로 하는 의존성을 알기 어렵고, `final` 필드를 사용할 수 없다.

Spring이 관리하지 않고 직접 생성한 객체에는 자동 주입이 일어나지 않는다.

```java
OrderService orderService = new OrderService();
```

`new`로 만든 객체는 일반적으로 컨테이너의 생성 및 후처리 과정을 통과하지 않으므로, 필드에 `@Autowired`가 있어도 Spring이 개입하지 않는다.

## 기본 후보 탐색: 타입

Spring은 기본적으로 주입 지점이 요구하는 타입을 기준으로 후보 Bean을 찾는다.

```java
public OrderService(PaymentService paymentService) {
    this.paymentService = paymentService;
}
```

다음 구현체 하나만 등록되어 있다면 그 Bean이 주입된다.

```java
@Component
public class KakaoPaymentService
        implements PaymentService {
}
```

하지만 동일한 타입의 구현체가 두 개라면 후보가 모호해진다.

```java
@Component
public class KakaoPaymentService
        implements PaymentService {
}
```

```java
@Component
public class CardPaymentService
        implements PaymentService {
}
```

어느 객체를 선택해야 하는지 결정할 수 없으면 일반적으로 `NoUniqueBeanDefinitionException`이 발생한다.

## `@Primary`

여러 후보 가운데 기본 선택 대상을 지정한다.

```java
@Primary
@Component
public class KakaoPaymentService
        implements PaymentService {
}
```

```java
@Component
public class CardPaymentService
        implements PaymentService {
}
```

별도의 한정 조건 없이 `PaymentService`를 요청하면 `@Primary`가 붙은 Bean이 우선 선택된다.

## `@Qualifier`

주입 지점에서 원하는 후보를 구체적으로 지정한다.

```java
@Component
@Qualifier("kakao")
public class KakaoPaymentService
        implements PaymentService {
}
```

```java
@Component
@Qualifier("card")
public class CardPaymentService
        implements PaymentService {
}
```

```java
@Service
public class OrderService {

    public OrderService(
            @Qualifier("card")
            PaymentService paymentService) {
    }
}
```

`@Primary`가 기본값을 지정한다면 `@Qualifier`는 특정 주입 지점에서 더 구체적인 선택 조건을 제공한다.

문자열 사용을 줄이기 위해 사용자 정의 한정 애노테이션을 만들 수도 있다.

```java
@Target({
    ElementType.TYPE,
    ElementType.FIELD,
    ElementType.PARAMETER,
    ElementType.METHOD
})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface CardPayment {
}
```

```java
@Component
@CardPayment
public class CardPaymentService
        implements PaymentService {
}
```

```java
public OrderService(
        @CardPayment PaymentService paymentService) {
}
```

## 이름을 이용한 후보 구분

여러 타입 후보가 있을 때 Spring은 일부 상황에서 주입 매개변수명이나 필드명을 Bean 이름과 비교하여 후보를 좁힐 수 있다.

```java
public OrderService(
        PaymentService kakaoPaymentService) {
}
```

그러나 의도가 중요한 선택은 이름 추론에만 의존하기보다 `@Qualifier`로 명시하는 편이 분명하다.

후보 선택 과정은 개념적으로 다음과 같이 이해할 수 있다.

```text
요구 타입으로 후보 검색
        ↓
Qualifier 조건 확인
        ↓
Primary 후보 확인
        ↓
이름 등 추가 조건 확인
        ↓
최종 Bean 선택 또는 예외
```

실제 세부 규칙은 주입 지점과 Spring 설정에 따라 더 복잡할 수 있다.

## 컬렉션 주입

동일한 인터페이스를 구현한 모든 Bean을 한 번에 주입할 수 있다.

```java
@Service
public class PaymentRouter {

    private final List<PaymentService> services;

    public PaymentRouter(
            List<PaymentService> services) {

        this.services = services;
    }
}
```

Map으로 받으면 일반적으로 Bean 이름이 키가 된다.

```java
public PaymentRouter(
        Map<String, PaymentService> services) {
    this.services = services;
}
```

```text
kakaoPaymentService -> KakaoPaymentService Bean
cardPaymentService  -> CardPaymentService Bean
```

컬렉션의 순서가 필요하면 `@Order` 또는 `Ordered`를 사용할 수 있다.

```java
@Component
@Order(1)
public class FirstPaymentHandler
        implements PaymentHandler {
}
```

## 설정값 주입

Spring은 객체뿐 아니라 외부 설정값도 주입할 수 있다.

```java
@Component
public class ApiClient {

    public ApiClient(
            @Value("${external.api.url}")
            String apiUrl) {
    }
}
```

관련 설정이 많다면 `@ConfigurationProperties`로 하나의 타입에 묶을 수 있다.

```java
@ConfigurationProperties(prefix = "payment")
public record PaymentProperties(
        String baseUrl,
        Duration timeout,
        int maxRetries
) {
}
```

```yaml
payment:
  base-url: https://payment.example.com
  timeout: 3s
  max-retries: 3
```

```java
@Component
public class PaymentClient {

    private final PaymentProperties properties;

    public PaymentClient(
            PaymentProperties properties) {

        this.properties = properties;
    }
}
```

