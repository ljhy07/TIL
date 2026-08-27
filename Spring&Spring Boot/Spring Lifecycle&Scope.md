## Singleton Bean의 일반적인 생명주기

Spring 컨테이너가 관리하는 Singleton Bean은 개념적으로 다음 과정을 거친다.

```text
Bean 정의 읽기
    ↓
Bean 인스턴스 생성
    ↓
의존관계 주입
    ↓
Aware 콜백
    ↓
초기화 전 BeanPostProcessor
    ↓
초기화 콜백
    ↓
초기화 후 BeanPostProcessor
    ↓
Bean 사용
    ↓
컨테이너 종료
    ↓
소멸 콜백
```

실제 내부 구현은 더 복잡하지만, 의존관계 주입이 완료된 뒤 초기화 콜백이 호출되고 컨테이너 종료 시 소멸 콜백이 호출된다는 흐름이 중요하다.

## 초기화와 소멸 콜백

`@PostConstruct`와 `@PreDestroy`를 사용할 수 있다.

```java
@Component
public class ExternalClient {

    private final ClientProperties properties;

    public ExternalClient(
            ClientProperties properties) {
        this.properties = properties;
    }

    @PostConstruct
    public void initialize() {
        // 의존관계 주입 후 초기화
    }

    @PreDestroy
    public void close() {
        // 컨테이너 종료 전 자원 정리
    }
}
```

`@PostConstruct` 시점에는 생성자 주입이 완료되었으므로 `properties`를 사용할 수 있다.

Java 설정에서는 초기화 및 소멸 메서드 이름을 지정할 수도 있다.

```java
@Bean(
    initMethod = "connect",
    destroyMethod = "disconnect"
)
public ExternalClient externalClient() {
    return new ExternalClient();
}
```

생성자는 객체의 필수 상태를 설정하는 곳이고 초기화 콜백은 의존관계 주입 후 필요한 준비 작업을 수행하는 곳으로 구분할 수 있다.

## Singleton 스코프

Spring의 기본 스코프는 Singleton이다.

```java
@Component
public class OrderService {
}
```

하나의 `ApplicationContext` 안에서 해당 Bean 정의에 대해 일반적으로 하나의 인스턴스를 생성하고 공유한다.

Spring Singleton은 전통적인 싱글턴 패턴과 구분해야 한다.

- JVM 전체에 반드시 하나라는 의미가 아니다.
- `ApplicationContext`가 여러 개면 컨텍스트별 인스턴스가 존재할 수 있다.
- private 생성자나 정적 `getInstance()`가 필요하지 않는다.
- 인스턴스의 생성과 보관을 컨테이너가 담당한다.

Singleton Bean은 여러 스레드에서 함께 사용될 수 있다. 따라서 요청마다 달라지는 데이터를 인스턴스 필드에 저장하면 동시성 문제가 발생할 수 있다.

```java
@Service
public class OrderService {

    private Long currentUserId;

    public void order(Long userId) {
        this.currentUserId = userId;
    }
}
```

서로 다른 요청이 동일한 Bean의 필드를 덮어쓸 수 있다. 요청 데이터는 일반적으로 매개변수와 지역 변수로 처리한다.

```java
public OrderResult order(Long userId) {
    return process(userId);
}
```

## Prototype 스코프

Prototype Bean은 컨테이너에 요청할 때마다 새 인스턴스를 생성한다.

```java
@Component
@Scope("prototype")
public class WorkUnit {
}
```

```java
WorkUnit first =
        context.getBean(WorkUnit.class);

WorkUnit second =
        context.getBean(WorkUnit.class);
```

`first`와 `second`는 서로 다른 객체입니다.

Spring은 Prototype Bean의 생성, 의존관계 주입, 초기화까지 처리하지만 일반적으로 Singleton과 같은 완전한 종료 생명주기를 관리하지는 않는다. Prototype 객체가 별도 정리 작업을 요구한다면 이를 사용하는 쪽에서 관리해야 할 수 있다.

## Web 스코프

웹 애플리케이션에서는 HTTP 요청이나 세션에 맞춘 스코프를 사용할 수 있다.

- `request`: HTTP 요청당 하나
- `session`: HTTP 세션당 하나
- `application`: `ServletContext`당 하나
- `websocket`: WebSocket 세션당 하나

```java
@Component
@RequestScope
public class RequestContext {
}
```

`RequestContext`는 각 HTTP 요청 범위에서 독립적인 인스턴스로 동작한다.

## 서로 다른 스코프의 주입 문제

Singleton Bean은 보통 컨테이너 초기화 과정에서 한 번 생성된다. 이때 Prototype Bean을 직접 주입하면 주입 시점에 생성된 하나의 객체가 Singleton 내부에 계속 보관될 수 있다.

```java
@Service
public class SingletonService {

    private final WorkUnit workUnit;

    public SingletonService(WorkUnit workUnit) {
        this.workUnit = workUnit;
    }
}
```

`WorkUnit`이 Prototype이어도 `SingletonService`가 메서드를 호출할 때마다 필드에 새 객체가 자동으로 대입되는 것은 아니다.

호출할 때마다 새 Bean이 필요하다면 `ObjectProvider`를 사용할 수 있다.

```java
@Service
public class SingletonService {

    private final ObjectProvider<WorkUnit> provider;

    public SingletonService(
            ObjectProvider<WorkUnit> provider) {

        this.provider = provider;
    }

    public void execute() {
        WorkUnit workUnit = provider.getObject();
        workUnit.run();
    }
}
```

## 스코프 프록시

수명이 긴 Bean에 수명이 짧은 Bean을 주입할 때 프록시를 사용할 수도 있다.

```java
@Component
@Scope(
    value = "prototype",
    proxyMode = ScopedProxyMode.TARGET_CLASS
)
public class WorkUnit {
}
```

이 경우 의존하는 Bean에는 실제 `WorkUnit` 대신 프록시가 주입된다. 메서드가 호출될 때 프록시가 현재 스코프에 맞는 실제 대상 객체를 찾아 호출을 전달한다.

```text
SingletonService
      ↓
WorkUnit 프록시
      ↓ 호출 시점에 대상 조회
현재 스코프의 WorkUnit
```

Request 스코프 Bean을 Singleton Bean에 주입하는 경우에도 같은 원리의 프록시가 사용될 수 있다.

