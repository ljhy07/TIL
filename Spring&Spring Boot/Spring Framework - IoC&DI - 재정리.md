## 의존성이란 무엇인가

객체 A가 자신의 작업을 수행하기 위해 객체 B를 사용한다면 A는 B에 의존한다.

```java
public class OrderService {

    private final PaymentService paymentService;

    public void order(long amount) {
        paymentService.pay(amount);
    }
}
```

여기서 `OrderService`는 주문을 처리하려면 `PaymentService`가 필요하다. 따라서 `OrderService`는 `PaymentService`에 의존한다.

객체들이 협력하려면 의존성은 반드시 존재한다. 문제는 의존성 자체가 아니라 객체가 의존 대상을 어떻게 얻는가이다.

## 객체가 의존 대상을 직접 생성하는 방식

```java
public class OrderService {

    private final PaymentService paymentService =
            new KakaoPaymentService();
}
```

이 코드에서 `OrderService`는 다음 두 가지 책임을 동시에 가진다.

- 주문을 처리한다.
- 사용할 결제 구현체를 결정하고 생성한다.

또한 `KakaoPaymentService`라는 구체 클래스와 객체 생성 방법을 직접 알고 있다. 다른 결제 구현체를 사용하려면 `OrderService`의 코드를 수정해야 한다.

## Dependency Injection

DI는 객체가 필요로 하는 의존 대상을 외부에서 전달하는 방식이다.

```java
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

이제 `OrderService`는 `PaymentService`를 직접 생성하지 않는다. 외부에서 전달된 객체를 사용한다.

순수 Java 코드로 표현하면 다음과 같다.

```java
PaymentService paymentService =
        new KakaoPaymentService();

OrderService orderService =
        new OrderService(paymentService);
```

Spring을 사용하지 않아도 DI는 가능하다. Spring DI는 이 객체 생성과 연결 작업을 Spring 컨테이너가 대신 수행하는 것이다.

## IoC와 제어의 역전

IoC는 Inversion of Control, 즉 제어의 역전이다.

일반적인 Java 프로그램에서는 애플리케이션 코드가 객체를 직접 생성하고 연결한다.

```java
PaymentService paymentService =
        new KakaoPaymentService();

OrderService orderService =
        new OrderService(paymentService);

orderService.order(10_000L);
```

Spring 애플리케이션에서는 컨테이너가 다음 작업을 담당한다.

- 객체 생성
- 객체 사이의 의존관계 연결
- 초기화와 종료 처리
- 객체의 보관과 조회
- 객체가 공유되는 범위 관리
- 후처리기 및 프록시 적용

객체 관리의 주도권이 애플리케이션 코드에서 컨테이너로 이동했기 때문에 제어가 역전되었다고 표현한다.

## IoC와 DI의 관계

IoC는 더 넓은 개념이고 DI는 IoC를 실현하는 대표적인 방법이다.

```text
IoC
└── 객체 관리의 제어권을 외부에 맡기는 원칙
    └── DI
        └── 필요한 의존 객체를 외부에서 주입하는 방식
```

Spring에서는 컨테이너가 객체를 생성하고, 해당 객체에 필요한 다른 객체를 찾아 전달함으로써 IoC를 구현한다.

## DI와 DIP의 차이

DI와 DIP는 서로 관련 있지만 다른 개념이다.

DI는 의존성을 외부에서 전달하는 구현 방식이다.

```java
public OrderService(
        KakaoPaymentService paymentService) {
    this.paymentService = paymentService;
}
```

위 코드도 의존 객체를 외부에서 받으므로 DI이다. 하지만 `OrderService`가 구체 구현체에 직접 의존한다.

DIP는 Dependency Inversion Principle, 즉 의존관계 역전 원칙이다. 상위 정책과 하위 구현이 구체 클래스가 아니라 추상화에 의존하도록 만드는 원칙이다.

```java
public interface PaymentService {
    void pay(long amount);
}
```

```java
public class KakaoPaymentService
        implements PaymentService {

    @Override
    public void pay(long amount) {
        // 카카오 결제 처리
    }
}
```

```java
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

의존 방향은 다음과 같다.

```text
OrderService ──> PaymentService <── KakaoPaymentService
```

DI를 사용하면 DIP를 적용하기 쉬워지지만, DI를 사용했다는 사실만으로 DIP까지 자동으로 만족하는 것은 아니다.

## Spring DI의 본질

Spring DI의 핵심은 애노테이션 자체가 아니다.

```text
비즈니스 객체
└── 자신이 수행할 작업에 집중한다.

Spring 컨테이너
└── 객체를 만들고 서로 연결한다.
```

`@Autowired`는 이 연결을 표현하는 방법 중 하나일 뿐이다. 중요한 것은 객체가 자신의 의존 객체를 직접 생성하지 않고 외부에서 전달받는 구조이다.