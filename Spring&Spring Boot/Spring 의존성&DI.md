## Spring의 의존성
#### **라이브러리 의존성**
`spring-web`, `spring-data-jpa`, `mysql-connector`처럼 Gradle/Maven에 추가하는 외부 라이브러리이다.

#### **객체 의존성**
`OrderService`가 `OrderRepository`나 `DiscountPolicy`를 필요로 하는 관계이다.

## DI **핵심 개념**
Dependency Injection은 **필요한 객체를 클래스가 직접 만들지 않고, 외부에서 넣어주는 방식**이다.

**DI 미사용**

```
public class OrderService {
    private final DiscountPolicy discountPolicy = new RateDiscountPolicy();
}
```

`OrderService`가 `RateDiscountPolicy`를 직접 생성하므로 강하게 결합된다. 나중에 `FixDiscountPolicy`로 바꾸려면 `OrderService` 코드를 수정해야 한다.

**DI 사용**

```
@Service
public class OrderService {
    private final DiscountPolicy discountPolicy;

    public OrderService(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
}
```

```
@Component
public class RateDiscountPolicy implements DiscountPolicy {
}
```

이제 `OrderService`는 구체 클래스가 아니라 `DiscountPolicy` 인터페이스에만 의존한다. 어떤 구현체를 넣을지는 Spring 컨테이너가 결정한다.

## **IoC와 DI의 관계**

Spring의 큰 원리는 **IoC**, Inversion of Control이다.

일반적인 코드에서는 객체가 직접 의존 객체를 만든다.

```
new RateDiscountPolicy()
```

하지만 Spring에서는 객체 생성과 연결을 Spring 컨테이너가 담당한다. 즉, 제어권이 개발자 코드에서 프레임워크로 넘어갑니다. 이 IoC를 구현하는 대표적인 방식이 **DI**이다.


## **DI 방식**

Spring에서는 주로 세 가지 방식으로 의존성을 주입한다.

```
// 1. 생성자 주입
public OrderService(OrderRepository orderRepository) {
    this.orderRepository = orderRepository;
}
```

```
// 2. Setter 주입
public void setOrderRepository(OrderRepository orderRepository) {
    this.orderRepository = orderRepository;
}
```

```
// 3. 필드 주입
@Autowired
private OrderRepository orderRepository;
```

**생성자 주입**을 가장 권장한다. 
이유는 의존성이 명확하고, `final`을 사용할 수 있고, 테스트하기 좋고, 필수 의존성을 객체 생성 시점에 강제할 수 있기 때문이다.

## **정리**

Spring DI는 객체 간 의존 관계를 Spring 컨테이너가 대신 연결해주는 방식이다. 클래스는 구체 구현체에 덜 묶이고, 코드 변경이 쉬워지며, 테스트도 편해진다.

공식 문서 기준으로는 Spring IoC 컨테이너가 Bean을 만들면서 생성자 인자, 팩토리 메서드 인자, 프로퍼티 등을 통해 의존성을 주입한다.