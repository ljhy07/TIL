한 줄 설명
> **Spring Bean**은 “Spring IoC 컨테이너가 생성하고 관리하는 객체”이다.

## Spring Bean이란?

Spring Bean은 Spring 컨테이너가 직접 생성, 조립, 관리하는 객체이다.

예를 들어 이런 클래스가 있다고 하면
```
@Component
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

`UserService`는 단순한 Java 클래스이다. 하지만 Spring이 이 객체를 생성하고, 필요한 의존성인 `UserRepository`를 넣어주고, 생명주기까지 관리하면 이 객체는 **Spring Bean**이다.

#### Spring Bean의 핵심 
> 객체의 생성과 의존관계 관리를 개발자가 직접 `new`로 하지 않고, Spring 컨테이너에게 맡긴다.

일반 Java 코드:
```
UserRepository repo = new UserRepository();
UserService service = new UserService(repo);
```

Spring 방식:
```
@Component
public class UserRepository {
}

@Component
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

Spring이 알아서:

1. `UserRepository` 객체 생성
2. `UserService` 객체 생성
3. `UserService` 생성자에 `UserRepository` 주입
4. 필요한 경우 프록시 생성
5. 초기화 콜백 실행
6. 애플리케이션 종료 시 소멸 콜백 실행

을 처리한다.

---

## Spring Bean 등록

### 컴포넌트 스캔

```
@Component
public class PaymentService {
}
```

`@Component` 계열 애너테이션도 Bean 등록 대상이다.

```
@Service
@Repository
@Controller
@RestController
@Configuration
```

이들은 내부적으로 `@Component` 성격을 가진다. 다만 의미가 다르다.

- `@Service`: 비즈니스 로직 계층
- `@Repository`: 데이터 접근 계층
- `@Controller`: MVC 컨트롤러
- `@RestController`: REST API 컨트롤러
- `@Configuration`: 설정 클래스

### `@Bean` 메서드

```
@Configuration
public class AppConfig {
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

이 방식은 보통 외부 라이브러리 객체를 Bean으로 등록할 때 많이 쓴다.

---

## Spring Bean의 Bean Definition과 객체

Spring 내부적으로는 Bean 하나를 단순히 객체로만 보지 않는다.

Spring은 먼저 Bean에 대한 메타데이터를 가지고 있다. 이를 `BeanDefinition`이라고 한다.

`BeanDefinition`에는 아래의 정보가 들어간다.

- 어떤 클래스를 만들 것인가
- Bean 이름은 무엇인가
- 스코프는 무엇인가
- 생성자 인자는 무엇인가
- 의존성은 무엇인가
- 초기화 메서드는 무엇인가
- 지연 로딩 여부
- 프록시 적용 여부

즉 Spring Bean은 단순히 “객체 하나”라기보다:

> Spring 컨테이너가 이해하고 관리하는 객체 정의와 그 정의로부터 만들어진 인스턴스

에 가깝다.

---

## Spring Bean의 생명주기

1. Bean 정의 로딩
2. Bean 인스턴스 생성
3. 의존성 주입
4. `Aware` 인터페이스 콜백 실행
5. BeanPostProcessor 전처리
6. 초기화 콜백 실행
7. BeanPostProcessor 후처리
8. Bean 사용
9. 컨테이너 종료 시 소멸 콜백 실행

예시:
```
@Component
public class MyBean {

    public MyBean() {
        System.out.println("생성자");
    }

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

여기서 중요한 점은 생성자 호출이 끝났다고 Bean이 완전히 준비된 것은 아니라는 점이다. 의존성 주입, 초기화 콜백, 프록시 적용까지 끝나야 실제로 애플리케이션에서 사용할 준비가 된다.

---

## Spring Bean의 스코프

Spring Bean은 기본적으로 `singleton`이다.

```
@Component
public class UserService {
}
```

이 Bean은 애플리케이션 컨텍스트 안에서 하나만 만들어진다.

주의할 점은 Spring의 singleton은 Java의 Singleton 패턴과 다르다.

Java Singleton 패턴:
```
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();

    private Singleton() {}

    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

Spring singleton:
> Spring 컨테이너 하나당 객체 인스턴스 하나

즉 JVM 전체에서 하나가 아니라, **ApplicationContext 하나 안에서 하나**이다.

다른 스코프도 있습니다.

- `singleton`: 컨테이너당 하나
- `prototype`: 요청할 때마다 새 객체
- `request`: HTTP 요청당 하나
- `session`: HTTP 세션당 하나
- `application`: ServletContext당 하나
- `websocket`: WebSocket 세션당 하나

예시:
```
@Scope("prototype")
@Component
public class TemporaryObject {
}
```