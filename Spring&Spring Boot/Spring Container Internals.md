## BeanDefinition

Spring은 Bean 객체를 만들기 전에 Bean을 어떻게 생성하고 관리할지 나타내는 메타데이터를 준비한다. 이 메타데이터의 중심이 `BeanDefinition`이다.

`BeanDefinition`에는 개념적으로 다음과 같은 정보가 들어간다.

- Bean 클래스 또는 생성 정보
- Bean 이름
- 스코프
- 생성자 인자와 프로퍼티 값
- 지연 초기화 여부
- 초기화 및 소멸 메서드
- 의존 대상
- Primary 여부
- 역할과 설명

설정 방식이 달라도 결과적으로 컨테이너가 이해할 수 있는 Bean 정의로 변환된다.

```text
@Component / @Bean / XML
          ↓
    BeanDefinition 등록
          ↓
       BeanFactory
          ↓
 Bean 생성 및 의존성 주입
```

컴포넌트 스캔은 대상 클래스를 검색하고 Bean 정의를 등록한다. `@Configuration` 처리기는 `@Bean` 메서드를 분석하여 Bean 정의를 등록한다.

## 생성자 주입의 내부 흐름

다음 클래스를 컨테이너가 생성한다고 가정한다.

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

컨테이너는 개념적으로 다음 과정을 수행한다.

1. `OrderService`의 Bean 정의를 찾는다.
2. 인스턴스 생성에 사용할 생성자를 결정한다.
3. 생성자 매개변수 타입과 애노테이션을 분석한다.
4. `PaymentService` 타입의 후보 Bean을 찾는다.
5. `OrderRepository` 타입의 후보 Bean을 찾는다.
6. 한정자와 우선순위 규칙으로 각 후보를 확정한다.
7. 필요한 Bean이 아직 없다면 먼저 생성한다.
8. 생성자를 호출하여 `OrderService` 인스턴스를 만든다.
9. 초기화 전후의 후처리를 실행한다.
10. 완성된 Singleton Bean을 컨테이너에 보관한다.

단순화하면 Spring이 다음 조립 코드를 대신 실행하는 것과 비슷하다.

```java
PaymentService paymentService =
        new KakaoPaymentService();

OrderRepository orderRepository =
        new JpaOrderRepository();

OrderService orderService =
        new OrderService(
                paymentService,
                orderRepository
        );
```

실제로는 여기에 스코프, 프록시, 생명주기 콜백, 후처리기, 순환 참조 처리 등이 추가된다.

## BeanFactoryPostProcessor

`BeanFactoryPostProcessor`는 일반 Bean 인스턴스가 본격적으로 생성되기 전에 Bean 정의와 컨테이너 설정에 개입할 수 있는 확장 지점이다.

```text
BeanDefinition 등록
        ↓
BeanFactoryPostProcessor 실행
        ↓
일반 Bean 인스턴스 생성
```

Bean 객체 자체가 아니라 객체를 만들기 위한 메타데이터를 다루는 단계라는 점이 중요하다. 설정 클래스 분석이나 프로퍼티 관련 처리 등 Spring 인프라의 여러 부분이 이 계열의 확장 지점과 연결된다.

## BeanPostProcessor

`BeanPostProcessor`는 Bean 인스턴스가 생성된 뒤 초기화 전후에 개입한다.

```java
public interface BeanPostProcessor {

    Object postProcessBeforeInitialization(
            Object bean,
            String beanName);

    Object postProcessAfterInitialization(
            Object bean,
            String beanName);
}
```

개념적인 실행 위치는 다음과 같다.

```text
Bean 인스턴스 생성
        ↓
의존관계 주입
        ↓
초기화 전 BeanPostProcessor
        ↓
초기화 콜백
        ↓
초기화 후 BeanPostProcessor
        ↓
사용 가능한 Bean
```

Bean 후처리기는 Spring의 중요한 확장 지점이며 다음 기능의 기반이 된다.

- `@Autowired` 처리
- `@PostConstruct` 처리
- AOP 프록시 생성
- 트랜잭션 프록시 적용
- 비동기 및 캐시 프록시 적용

엄밀히 말하면 `@Autowired` 애노테이션이 스스로 의존성을 주입하는 것은 아니다. 등록된 후처리기가 Bean의 주입 지점을 찾고 컨테이너에서 후보를 조회하여 값을 설정한다.

## `@Configuration` 클래스의 프록시 동작

다음 설정에서는 `orderService()`가 `paymentService()` 메서드를 직접 호출한다.

```java
@Configuration
public class AppConfig {

    @Bean
    public PaymentService paymentService() {
        return new KakaoPaymentService();
    }

    @Bean
    public OrderService orderService() {
        return new OrderService(paymentService());
    }
}
```

일반 Java 메서드 호출이라면 `paymentService()`를 호출할 때마다 새 객체를 만들 수 있다. 전통적인 전체 `@Configuration` 처리에서는 Spring이 설정 클래스를 확장한 프록시를 만들어 `@Bean` 메서드 호출을 가로챌 수 있다.

Singleton Bean이 이미 만들어져 있다면 프록시는 메서드 본문에서 새 객체를 생성하는 대신 컨테이너가 관리하는 Bean을 반환한다.

의존관계를 더 명시적으로 표현하려면 매개변수 주입을 사용할 수 있다.

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

`@Configuration(proxyBeanMethods = false)`에서는 설정 클래스의 `@Bean` 메서드 간 직접 호출이 컨테이너 조회로 가로채지지 않을 수 있다.

```java
@Configuration(proxyBeanMethods = false)
public class AppConfig {
}
```

이 모드에서도 `@Bean` 메서드의 매개변수로 다른 Bean을 받으면 컨테이너가 정상적으로 의존성을 해결한다.

## FactoryBean과 BeanFactory

이름이 비슷하지만 역할은 완전히 다르다.

`BeanFactory`는 Bean을 관리하는 컨테이너 인터페이스이다.

`FactoryBean<T>`는 특정 객체를 생성하는 로직을 캡슐화한 Bean이다.

```java
@Component
public class ClientFactoryBean
        implements FactoryBean<ApiClient> {

    @Override
    public ApiClient getObject() {
        return new ApiClient();
    }

    @Override
    public Class<?> getObjectType() {
        return ApiClient.class;
    }
}
```

일반적으로 Bean 이름을 조회하면 `FactoryBean` 자체가 아니라 팩토리가 만든 객체를 얻는다.

```java
context.getBean("clientFactoryBean");
```

팩토리 객체 자체를 조회하려면 `&` 접두사를 사용한다.

```java
context.getBean("&clientFactoryBean");
```

```text
BeanFactory
└── 여러 Bean의 생성과 관리를 담당하는 컨테이너

FactoryBean<T>
└── 특정 타입 T의 생성 방법을 제공하는 하나의 Bean
```

