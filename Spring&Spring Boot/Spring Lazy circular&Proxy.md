## 선택적 의존성

모든 의존성이 반드시 존재해야 하는 것은 아니다. `Optional<T>`를 주입받으면 해당 타입의 Bean이 없을 때 `Optional.empty()`를 받을 수 있다.

```java
public OrderService(
        Optional<CouponService> couponService) {
}
```

선택적 기능을 표현할 수 있지만, 핵심 의존성을 무분별하게 `Optional`로 만들면 객체가 실제로 무엇을 필요로 하는지 불분명해질 수 있다.

## ObjectProvider

`ObjectProvider<T>`는 Spring 컨테이너에서 대상 Bean을 필요할 때 가져올 수 있게 해준다.

```java
@Service
public class OrderService {

    private final ObjectProvider<CouponService> provider;

    public OrderService(
            ObjectProvider<CouponService> provider) {

        this.provider = provider;
    }

    public void order() {
        CouponService couponService =
                provider.getIfAvailable();

        if (couponService != null) {
            couponService.apply();
        }
    }
}
```

주요 메서드는 다음과 같다.

```java
provider.getObject();
provider.getIfAvailable();
provider.getIfUnique();
provider.stream();
provider.orderedStream();
```

직접 생성자 주입은 Bean이 만들어질 때 의존 객체를 확정한다. `ObjectProvider`는 실제 사용 시점까지 조회를 미룰 수 있고, Prototype처럼 호출마다 새 객체를 찾아야 하는 상황에도 사용할 수 있다.

## `@Lazy`

```java
public OrderService(
        @Lazy ReportService reportService) {
}
```

주입 지점에 `@Lazy`를 사용하면 실제 대상 대신 지연 프록시가 들어갈 수 있다. 의존 객체를 즉시 생성하지 않고 처음 사용하는 시점까지 실제 대상 조회와 생성을 미룬다.

```text
OrderService 생성
      ↓
ReportService 지연 프록시 주입
      ↓
처음 메서드를 호출할 때
      ↓
실제 ReportService 조회 또는 생성
```

## 순환 의존성

두 객체가 서로를 필요로 하면 순환 의존성이 생긴다.

```java
@Service
public class ServiceA {

    public ServiceA(ServiceB serviceB) {
    }
}
```

```java
@Service
public class ServiceB {

    public ServiceB(ServiceA serviceA) {
    }
}
```

생성 순서는 끝나지 않는 고리가 된다.

```text
ServiceA 생성 시도
    ↓
ServiceB가 필요함
    ↓
ServiceB 생성 시도
    ↓
ServiceA가 필요함
    ↓
ServiceA가 아직 완성되지 않음
```

생성자 순환 의존성에서는 어느 객체도 먼저 완성할 수 없으므로 생성에 실패한다.

Setter나 필드 주입으로 구성된 일부 Singleton 순환 참조는 조기 참조를 사용해 처리될 수 있지만 항상 가능한 것은 아니다. 프록시와 후처리가 결합되면 상황이 더 복잡해지고, Spring Boot의 설정에 따라 순환 참조 자체가 거부될 수도 있다.

`@Lazy`를 한쪽 주입 지점에 사용하면 즉시 실제 객체를 요구하지 않으므로 생성 고리를 늦출 수 있다.

```java
public ServiceA(@Lazy ServiceB serviceB) {
}
```

이는 객체 생성 시점의 고리를 우회하는 방법이지, 두 객체가 서로 강하게 연결되어 있다는 사실을 제거하는 것은 아니다.

## Spring AOP 프록시와 DI

Spring에서 주입받은 객체는 원래 클래스의 인스턴스가 아니라 프록시일 수 있다.

```java
@Service
public class PaymentService {

    @Transactional
    public void pay() {
        // 데이터베이스 작업
    }
}
```

`@Transactional` 처리를 위해 Spring은 개념적으로 다음 구조를 구성할 수 있다.

```text
호출자
   ↓
PaymentService 프록시
   ↓
트랜잭션 시작
   ↓
실제 PaymentService.pay()
   ↓
커밋 또는 롤백
```

다른 Bean에 주입되는 객체는 실제 `PaymentService` 대신 이 프록시가 될 수 있다. 프록시는 대상과 같은 인터페이스를 구현하거나 대상 클래스를 상속하여 호출을 가로챈다.

대표적인 프록시 방식은 다음과 같다.

- 인터페이스 기반 동적 프록시
- 클래스 상속 기반 프록시

이 때문에 `paymentService.getClass()`의 결과가 원래 클래스와 정확히 같지 않을 수 있다.

## Self-invocation

프록시 기반 기능은 호출이 프록시를 통과해야 적용된다.

```java
@Service
public class OrderService {

    public void createOrder() {
        saveOrder();
    }

    @Transactional
    public void saveOrder() {
        // 데이터베이스 작업
    }
}
```

외부 객체가 `OrderService`를 호출할 때는 프록시를 통과할 수 있다. 그러나 `createOrder()` 안에서 실행하는 `saveOrder()` 호출은 일반적으로 현재 객체의 내부 호출이다.

```text
외부 호출
   ↓
OrderService 프록시
   ↓
실제 createOrder()
   ↓
this.saveOrder()
```

마지막 호출은 프록시를 다시 거치지 않기 때문에 `saveOrder()`에만 선언한 `@Transactional`이 기대한 방식으로 적용되지 않을 수 있다. 이 현상을 self-invocation 문제라고 부른다.

중요한 것은 애노테이션이 붙은 메서드인지뿐 아니라 그 메서드 호출이 Spring 프록시의 경계를 통과하는지이다.

## 프록시와 실제 객체의 관계

프록시가 적용된 Bean을 이해할 때 다음 세 대상을 구분하면 좋다.

```text
Bean을 사용하는 호출자
        ↓
컨테이너가 노출한 프록시
        ↓
실제 작업을 수행하는 대상 객체
```

DI 관점에서 컨테이너는 단순히 원본 객체만 연결하는 것이 아니다. Bean 후처리 결과 만들어진 프록시를 최종 Bean으로 노출하고, 그 프록시를 다른 Bean에 주입할 수도 있다.

따라서 Spring DI를 깊게 이해하려면 다음 흐름을 함께 봐야 한다.

```text
BeanDefinition
    ↓
객체 생성과 의존성 주입
    ↓
초기화 및 BeanPostProcessor
    ↓
필요하면 프록시 생성
    ↓
최종 Bean을 컨테이너에 노출
    ↓
다른 Bean에 주입
```

