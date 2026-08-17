Spring 컨테이너가 Bean 인스턴스를 **몇 개 생성하고, 얼마나 오래 유지하며, 어디까지 공유할지** 결정하는 규칙입니다.

예를 들어 같은 `OrderService`를 주입받았을 때:

- 애플리케이션 전체가 하나의 객체를 공유할 수도 있다.
- 주입하거나 조회할 때마다 새 객체를 만들 수도 있다.
- HTTP 요청마다 별도의 객체를 사용할 수도 있다.

## 주요 Bean Scope 한눈에 보기

| Scope         | 인스턴스 생성 기준           | 생명주기                  | 주 용도                     |
| ------------- | -------------------- | --------------------- | ------------------------ |
| `singleton`   | Spring 컨테이너당 1개      | 컨테이너 시작부터 종료까지        | 일반적인 Service, Repository |
| `prototype`   | 조회할 때마다 새로 생성        | 생성과 초기화까지만 Spring이 관리 | 상태를 가지는 단기 객체            |
| `request`     | HTTP 요청당 1개          | 요청 시작부터 종료까지          | 요청별 컨텍스트                 |
| `session`     | HTTP 세션당 1개          | 세션 생성부터 만료까지          | 장바구니, 로그인 세션 정보          |
| `application` | `ServletContext`당 1개 | 웹 애플리케이션 생명주기         | 웹 앱 전체 공유 상태             |
| `websocket`   | WebSocket 세션당 1개     | WebSocket 연결 동안       | 연결별 상태                   |

가장 중요한 것은 `singleton`, `prototype`, `request`, `session`이다.

---

# Singleton Scope

Spring Bean의 기본 Scope이다. 별도로 Scope를 지정하지 않으면 `singleton`으로 등록된다.

```
@Service
public class OrderService {
}
```

위 코드는 다음과 같은 의미이다.

```
@Service
@Scope("singleton")
public class OrderService {
}
```

컨테이너에서 같은 Bean을 여러 번 조회해도 동일한 인스턴스를 반환한다.

```
OrderService service1 = context.getBean(OrderService.class);
OrderService service2 = context.getBean(OrderService.class);

System.out.println(service1 == service2); // true
```

## 생성 시점

일반적인 singleton Bean은 Spring 컨테이너가 시작될 때 미리 생성된다.

```
@Component
public class SingletonBean {

    public SingletonBean() {
        System.out.println("SingletonBean 생성");
    }
}
```

필요할 때 생성하도록 `@Lazy`를 사용할 수도 있다.

```
@Component
@Lazy
public class LazyBean {
}
```

## 장점

- 객체를 반복해서 만들지 않아 효율적
- 의존성 관리가 단순함
- 대부분의 Service와 Repository에 적합함

## 가장 중요한 주의점: Thread Safety

Singleton Bean 하나를 여러 요청과 스레드가 동시에 공유한다. 따라서 요청별 데이터를 인스턴스 필드에 저장하면 안 된다.

잘못된 예:

```
@Service
public class PriceService {

    private int currentPrice;

    public int calculate(int price) {
        this.currentPrice = price;

        // 다른 스레드가 currentPrice를 변경할 수 있음
        return currentPrice;
    }
}
```

동시에 다음 요청이 들어올 수 있다.

```
Thread A: currentPrice = 10,000
Thread B: currentPrice = 20,000
Thread A: currentPrice 반환 → 20,000이 반환될 수 있음
```

안전한 형태:

```
@Service
public class PriceService {

    public int calculate(int price) {
        int currentPrice = price;
        return currentPrice;
    }
}
```

일반적으로 singleton Bean은 다음과 같이 설계한다.

- 변경 가능한 요청 상태를 필드에 저장하지 않음
- 지역 변수 사용
- 불변 객체 사용
- 공유 상태가 꼭 필요하면 동시성 제어 사용
- 사용자별 상태는 DB, 세션, 캐시 등에 저장

Spring singleton은 디자인 패턴의 전역 Singleton과도 조금 다르다. Spring singleton은 **Spring 컨테이너와 Bean 정의를 기준으로 하나**이다. 컨테이너가 여러 개라면 같은 타입의 singleton 인스턴스도 여러 개 존재할 수 있다.

---

# Prototype Scope

`prototype`은 Spring 컨테이너에서 Bean을 요청할 때마다 새로운 인스턴스를 생성한다.

```
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class PrototypeBean {
}
```

문자열로도 지정할 수 있다.

```
@Component
@Scope("prototype")
public class PrototypeBean {
}
```

조회 결과:

```
PrototypeBean bean1 = context.getBean(PrototypeBean.class);
PrototypeBean bean2 = context.getBean(PrototypeBean.class);

System.out.println(bean1 == bean2); // false
```

## 생명주기에서 중요한 점

Spring은 prototype Bean의 다음 과정까지만 관리한다.

1. 객체 생성
2. 의존성 주입
3. 초기화 콜백 실행
4. 호출자에게 반환

그 이후의 소멸 과정은 관리하지 않다.

```
@Component
@Scope("prototype")
public class PrototypeBean {

    @PostConstruct
    public void init() {
        System.out.println("초기화");
    }

    @PreDestroy
    public void destroy() {
        System.out.println("소멸");
    }
}
```

`@PostConstruct`는 실행되지만, 컨테이너가 종료될 때 `@PreDestroy`는 일반적으로 자동 실행되지 않는다. prototype Bean이 파일이나 네트워크 연결 같은 자원을 사용한다면 호출자가 직접 정리해야 한다.

## 언제 사용하는가?

- 각 작업이 독립적인 상태를 가져야 할 때
- 객체의 사용 시간이 짧을 때
- 작업 단위 객체를 매번 새로 만들어야 할 때

단순히 DTO를 만들기 위해 prototype Bean을 사용하는 것은 보통 과도한 설계이다. DTO는 일반적으로 `new`로 생성해도 충분하다.

---

# Singleton Bean에 Prototype Bean을 주입하면 생기는 문제

```
@Component
@Scope("prototype")
public class PrototypeBean {
}
```

```
@Service
public class SingletonService {

    private final PrototypeBean prototypeBean;

    public SingletonService(PrototypeBean prototypeBean) {
        this.prototypeBean = prototypeBean;
    }

    public PrototypeBean getPrototypeBean() {
        return prototypeBean;
    }
}
```

다음 코드는 새로운 prototype Bean을 받을 것 같지만 그렇지 않는다.

```
PrototypeBean bean1 = singletonService.getPrototypeBean();
PrototypeBean bean2 = singletonService.getPrototypeBean();

System.out.println(bean1 == bean2); // true
```

이유는 `SingletonService`가 생성되는 시점에 `PrototypeBean` 하나가 주입되고, singleton 내부에 그 참조가 계속 보관되기 때문이다.

`prototype`은 **필드를 사용할 때마다 새로 생성한다**는 뜻이 아니라, **Spring 컨테이너에 조회를 요청할 때마다 새로 생성한다**는 뜻이다.

## ObjectProvider 해결 방법 

가장 실용적인 방법이다.

```
@Service
public class SingletonService {

    private final ObjectProvider<PrototypeBean> provider;

    public SingletonService(ObjectProvider<PrototypeBean> provider) {
        this.provider = provider;
    }

    public PrototypeBean createPrototypeBean() {
        return provider.getObject();
    }
}
```

```
PrototypeBean bean1 = singletonService.createPrototypeBean();
PrototypeBean bean2 = singletonService.createPrototypeBean();

System.out.println(bean1 == bean2); // false
```

## `jakarta.inject.Provider` 해결 방법 

```
@Service
public class SingletonService {

    private final Provider<PrototypeBean> provider;

    public SingletonService(Provider<PrototypeBean> provider) {
        this.provider = provider;
    }

    public PrototypeBean createPrototypeBean() {
        return provider.get();
    }
}
```

## Scoped Proxy 해결 방법

```
@Component
@Scope(
    value = ConfigurableBeanFactory.SCOPE_PROTOTYPE,
    proxyMode = ScopedProxyMode.TARGET_CLASS
)
public class PrototypeBean {
}
```

singleton에는 실제 prototype Bean 대신 프록시가 주입된다. 프록시를 통해 메서드를 호출할 때 실제 Bean을 조회한다.

다만 prototype 프록시는 메서드 호출마다 새로운 대상이 선택될 수 있기 때문에, 동일 객체의 상태가 여러 메서드 호출에 걸쳐 유지될 것이라고 생각하면 혼란이 생길 수 있다. 명시적으로 새 인스턴스를 얻는 `ObjectProvider`가 이해하기 쉬운 경우가 많다.

---

# Request Scope

HTTP 요청 하나당 Bean 인스턴스 하나를 생성한다.

```
@Component
@RequestScope
public class RequestContext {

    private String requestId;

    public String getRequestId() {
        return requestId;
    }

    public void setRequestId(String requestId) {
        this.requestId = requestId;
    }
}
```

다음과 같이 작성한 것과 유사하다.

```
@Component
@Scope(
    value = WebApplicationContext.SCOPE_REQUEST,
    proxyMode = ScopedProxyMode.TARGET_CLASS
)
public class RequestContext {
}
```

같은 HTTP 요청 안에서는 같은 객체가 사용되고, 다른 요청에서는 새로운 객체가 사용된다.

```
HTTP 요청 A
  Controller → RequestContext A
  Service    → RequestContext A

HTTP 요청 B
  Controller → RequestContext B
  Service    → RequestContext B
```

## 사용 사례

- 요청 ID
- 사용자 인증 관련 요청 정보
- 요청 시작 시간
- 감사 로그용 컨텍스트
- 요청 내에서 여러 계층이 공유할 데이터

예:

```
@Component
@RequestScope
public class RequestInfo {

    private final String requestId = UUID.randomUUID().toString();
    private final long startTime = System.currentTimeMillis();

    public String getRequestId() {
        return requestId;
    }

    public long getStartTime() {
        return startTime;
    }
}
```

```
@RestController
public class OrderController {

    private final RequestInfo requestInfo;

    public OrderController(RequestInfo requestInfo) {
        this.requestInfo = requestInfo;
    }

    @GetMapping("/orders")
    public String orders() {
        return requestInfo.getRequestId();
    }
}
```

`@RequestScope`에는 기본적으로 클래스 기반 scoped proxy 설정이 포함되어 있기 때문에 singleton Controller에 주입할 수 있다.

## 주의점

HTTP 요청과 연결되지 않은 곳에서 request-scoped Bean을 실제로 사용하면 예외가 발생한다.

예:

- 스케줄러
- 애플리케이션 시작 이벤트
- 별도 비동기 스레드
- 일반 단위 테스트
- 메시지 큐 소비자

대표적인 오류:

```
Scope 'request' is not active for the current thread
```

특히 `@Async`를 호출하면 작업이 다른 스레드에서 실행되므로 원래 요청의 request scope가 자동으로 전달되지 않다. 필요한 값만 명시적으로 비동기 메서드 인자로 넘기는 것이 안전하다.

---

# Session Scope

HTTP 세션 하나당 하나의 Bean을 생성한다.

```
@Component
@SessionScope
public class ShoppingCart {

    private final List<Long> productIds = new ArrayList<>();

    public void addProduct(Long productId) {
        productIds.add(productId);
    }

    public List<Long> getProductIds() {
        return List.copyOf(productIds);
    }
}
```

동작 방식:

```
사용자 세션 A → ShoppingCart A
사용자 세션 B → ShoppingCart B
```

같은 사용자가 여러 요청을 보내도 세션이 유지되는 동안 같은 Bean을 사용한다.

## 사용 사례

- 장바구니
- 다단계 입력 폼의 중간 데이터
- 세션별 UI 설정
- 임시 사용자 상태

## 주의점

Session scope에는 몇 가지 운영상 부담이 있다.

### 메모리 사용

동시 사용자 수만큼 객체가 만들어질 수 있다. 큰 데이터나 무제한 컬렉션을 보관하면 서버 메모리를 많이 사용한다.

### 서버 확장

서버가 여러 대라면 요청이 다른 서버로 전달될 수 있다.

```
첫 번째 요청 → Server A의 Session Bean
두 번째 요청 → Server B에는 해당 상태가 없음
```

대응 방법:

- Sticky Session 사용
- Spring Session과 Redis 같은 공유 세션 저장소 사용
- 중요한 상태는 DB나 분산 캐시에 저장

### 직렬화

세션을 Redis 등에 저장하거나 서버 간 복제한다면 세션 객체가 직렬화 가능해야 할 수 있다.

### 보안

세션에 비밀번호, 토큰 원문, 과도한 개인정보 등을 저장하지 않는 것이 좋다.

---

# Application Scope

`ServletContext` 하나당 Bean 하나를 생성한다.

```
@Component
@ApplicationScope
public class ApplicationStatistics {
}
```

singleton과 비슷해 보이지만 기준이 다르다.

- `singleton`: Spring `ApplicationContext`당 하나
- `application`: 웹 `ServletContext`당 하나

대부분의 일반적인 Spring Boot 웹 애플리케이션에서는 두 Scope가 거의 같은 것처럼 동작한다. 특별히 `ServletContext` 수준의 생명주기가 필요하지 않다면 보통 singleton을 사용한다.

---

# WebSocket Scope

WebSocket 세션 하나당 Bean 하나를 생성한다.

```
@Component
@Scope("websocket")
public class WebSocketSessionState {
}
```

WebSocket 연결별로 상태를 관리할 때 사용한다.

예:

- 연결별 사용자 상태
- 구독 정보
- 연결 중 누적되는 임시 데이터

일반적인 REST API에서는 거의 사용하지 않는다.

---

# Scoped Proxy란?

Spring 애플리케이션에서는 singleton Bean이 request 또는 session Bean을 주입받는 일이 많다.

```
@Service
public class OrderService {

    private final RequestInfo requestInfo;

    public OrderService(RequestInfo requestInfo) {
        this.requestInfo = requestInfo;
    }
}
```

문제는 두 Bean의 생명주기가 다르다는 것이다.

```
OrderService: 애플리케이션 시작 시 생성
RequestInfo: 각 HTTP 요청이 시작될 때 생성
```

`OrderService`를 만드는 시점에는 실제 `RequestInfo`가 아직 존재하지 않는다. 그래서 Spring은 실제 Bean 대신 프록시 객체를 주입한다.

```
OrderService
    ↓ 참조
RequestInfo 프록시
    ↓ 메서드가 호출되는 현재 요청을 확인
현재 요청의 실제 RequestInfo
```

## 프록시 방식

```
@Scope(
    value = "request",
    proxyMode = ScopedProxyMode.TARGET_CLASS
)
```

`TARGET_CLASS`는 CGLIB 기반 클래스 프록시를 사용한다.

인터페이스 기반으로 프록시를 만들려면:

```
@Scope(
    value = "request",
    proxyMode = ScopedProxyMode.INTERFACES
)
```

대부분의 애플리케이션에서는 `@RequestScope`, `@SessionScope`, `@ApplicationScope`처럼 준비된 애노테이션을 쓰는 것이 편리합니다.

---

# Scope와 Bean 생명주기

## Singleton

```
컨테이너 시작
 → Bean 생성
 → 의존성 주입
 → @PostConstruct
 → 애플리케이션에서 사용
 → @PreDestroy
 → 컨테이너 종료
```

## Prototype

```
Bean 조회
 → Bean 생성
 → 의존성 주입
 → @PostConstruct
 → 호출자에게 반환
 → 이후 Spring이 소멸을 관리하지 않음
```

## Request

```
HTTP 요청 시작
 → Bean 생성
 → 요청 중 사용
 → 요청 종료
 → Bean 소멸
```

## Session

```
HTTP 세션 생성 또는 첫 사용
 → Bean 생성
 → 여러 요청에서 사용
 → 세션 만료/무효화
 → Bean 소멸
```

---

# `@Bean` 메서드에서 Scope 지정하기

컴포넌트 클래스뿐 아니라 설정 클래스에서도 지정할 수 있다.

```
@Configuration
public class AppConfig {

    @Bean
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public TaskContext taskContext() {
        return new TaskContext();
    }
}
```

Request scope 예:

```
@Configuration
public class WebConfig {

    @Bean
    @RequestScope
    public RequestInfo requestInfo(HttpServletRequest request) {
        return new RequestInfo(request.getRequestURI());
    }
}
```

---

# `@Configuration` 내부 `@Bean` 호출과 Scope

다음 코드는 겉으로 보면 Java 메서드를 직접 호출하는 것처럼 보인다.

```
@Configuration
public class AppConfig {

    @Bean
    public Client client() {
        return new Client(connection());
    }

    @Bean
    public Connection connection() {
        return new Connection();
    }
}
```

일반적인 `@Configuration`은 Spring이 프록시로 처리하기 때문에 `connection()`을 여러 번 호출해도 singleton Scope가 지켜진다.

그러나 다음처럼 `proxyBeanMethods = false`를 사용하면 일반 Java 메서드 호출처럼 동작할 수 있다.

```
@Configuration(proxyBeanMethods = false)
public class AppConfig {
}
```

이 모드에서는 `@Bean` 메서드끼리 직접 호출하는 대신 파라미터 주입을 사용하는 편이 안전하다.

```
@Configuration(proxyBeanMethods = false)
public class AppConfig {

    @Bean
    public Client client(Connection connection) {
        return new Client(connection);
    }

    @Bean
    public Connection connection() {
        return new Connection();
    }
}
```

---

# 사용자 정의 Scope

Spring은 커스텀 Scope도 만들 수 있다.

개념적으로 `Scope` 인터페이스를 구현한다.

```
public class CustomScope implements Scope {

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        // 현재 커스텀 컨텍스트에서 객체 조회
        // 없으면 objectFactory.getObject()로 생성
        return null;
    }

    @Override
    public Object remove(String name) {
        return null;
    }

    @Override
    public void registerDestructionCallback(
            String name,
            Runnable callback
    ) {
    }

    @Override
    public Object resolveContextualObject(String key) {
        return null;
    }

    @Override
    public String getConversationId() {
        return null;
    }
}
```

그리고 `CustomScopeConfigurer` 등을 사용해 등록한다.

실제 활용 예:

- Tenant별 Scope
- Job 실행별 Scope
- 배치 Step별 Scope
- 사용자 정의 메시지 처리 단위 Scope

다만 커스텀 Scope는 생명주기, 동시성, 소멸 콜백, 컨텍스트 전달 등을 모두 제대로 설계해야 하므로 명확한 필요가 있을 때 사용하는 것이 좋다.

Spring Batch에서는 자체적으로 `job`, `step` Scope를 제공한다.