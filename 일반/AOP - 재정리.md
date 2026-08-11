# AOP 핵심

AOP(Aspect-Oriented Programming, 관점 지향 프로그래밍)는 여러 기능에 반복해서 등장하는 공통 로직을 핵심 비즈니스 로직에서 분리하는 프로그래밍 방식이다.

특히 Java/Spring에서는 다음과 같은 기능에 AOP가 많이 사용된다.

- 로그 기록
- 실행 시간 측정
- 트랜잭션 처리
- 권한 검사
- 예외 기록
- 모니터링

Spring의 `@Transactional`도 대표적인 AOP 활용 사례인다.

---

## AOP가 필요한 이유

예를 들어 주문 서비스의 실행 시간을 측정해야 한다고 생각해 보면

```
public void createOrder() {
    long start = System.currentTimeMillis();

    // 주문 생성 로직
    validateOrder();
    saveOrder();

    long end = System.currentTimeMillis();
    System.out.println("실행 시간: " + (end - start));
}
```

회원 가입에도 같은 기능이 필요하다면 비슷한 코드를 또 작성해야 한다.

```
public void signUp() {
    long start = System.currentTimeMillis();

    // 회원 가입 로직
    validateMember();
    saveMember();

    long end = System.currentTimeMillis();
    System.out.println("실행 시간: " + (end - start));
}
```

이 방식에는 문제가 있다.

- 같은 코드가 여러 곳에서 반복된다.
- 핵심 로직과 부가 기능이 섞인다.
- 로깅 방식을 바꾸면 여러 파일을 수정해야 한다.
- 실수로 일부 메서드에 로깅을 빠뜨릴 수 있다.
- 테스트와 유지보수가 어려워진다.

AOP를 사용하면 실행 시간 측정 코드를 별도의 클래스로 분리할 수 있다.

```
@Aspect
@Component
public class ExecutionTimeAspect {

    @Around("@annotation(MeasureTime)")
    public Object measure(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();

        try {
            return joinPoint.proceed();
        } finally {
            long end = System.currentTimeMillis();

            System.out.println(
                joinPoint.getSignature().getName()
                    + " 실행 시간: "
                    + (end - start)
            );
        }
    }
}
```

비즈니스 코드에는 적용 여부만 표시한다.

```
@MeasureTime
public void createOrder() {
    validateOrder();
    saveOrder();
}
```

핵심 아이디어는 다음과 같다.

> 비즈니스 코드가 “무엇을 할지”에 집중하도록 만들고, 여러 곳에 공통으로 적용되는 부가 기능을 별도로 분리한다.

---

# 핵심 용어

| 용어         | 의미                      | 쉽게 표현하면                 |
| ---------- | ----------------------- | ----------------------- |
| Aspect     | 공통 기능을 모아 놓은 단위         | 로깅 담당 클래스               |
| Advice     | 실제로 실행되는 공통 로직          | 로그를 남기는 메서드             |
| Join Point | 공통 로직이 실행될 수 있는 지점      | 메서드 실행 시점               |
| Pointcut   | Advice를 적용할 대상을 선택하는 조건 | 어떤 메서드에 적용할지            |
| Target     | 실제 비즈니스 객체              | `OrderService` 객체       |
| Proxy      | Target을 감싸는 대리 객체       | Target 호출을 중간에서 가로채는 객체 |
| Weaving    | Target에 Aspect를 연결하는 과정 | 공통 기능을 적용하는 과정          |

Spring AOP를 이해할 때는 다음 흐름으로 보면 된다.

```
호출자
  ↓
Proxy
  ├─ 공통 기능 실행
  ├─ 실제 Target 메서드 실행
  └─ 공통 기능 실행
```

예를 들어 다음 코드를 호출한다고 해 보면

```
orderService.createOrder();
```

실제로는 대략 이런 과정이 일어난다.

```
1. 호출자가 Proxy의 createOrder()를 호출
2. Proxy가 로그 또는 트랜잭션 로직을 실행
3. Proxy가 실제 OrderService의 createOrder()를 호출
4. 실행 결과가 Proxy로 반환
5. Proxy가 추가 로직을 실행
6. 최종 결과가 호출자에게 반환
```

---

## Aspect

Aspect는 하나의 공통 관심사를 담당하는 클래스이다.

예를 들어 다음은 실행 시간을 측정하는 Aspect이다.

```
@Aspect
@Component
public class ExecutionTimeAspect {

    @Around("@annotation(MeasureTime)")
    public Object measure(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();

        try {
            return joinPoint.proceed();
        } finally {
            long end = System.currentTimeMillis();
            System.out.println("실행 시간: " + (end - start));
        }
    }
}
```

- `@Aspect`: 이 클래스가 AOP 클래스임을 나타낸다.
- `@Component`: Spring Bean으로 등록한다.
- `@Around`: 대상 메서드의 실행 전후에 개입한다.
- `joinPoint.proceed()`: 실제 대상 메서드를 실행한다.

---

## Advice

Advice는 특정 시점에 실행할 공통 로직이다.

Spring AOP에서 자주 사용하는 Advice는 다음과 같다.

### `@Before`

대상 메서드가 실행되기 전에 동작한다.

```
@Before("execution(* com.example.service..*(..))")
public void before() {
    System.out.println("메서드 실행 전");
}
```

주로 다음과 같은 용도로 사용한다.

- 요청 정보 기록
- 간단한 권한 확인
- 메서드 호출 로그 기록

---

### `@AfterReturning`

대상 메서드가 정상적으로 값을 반환한 뒤 실행된다.

```
@AfterReturning(
    pointcut = "execution(* com.example.service..*(..))",
    returning = "result"
)
public void afterReturning(Object result) {
    System.out.println("반환값: " + result);
}
```

예외가 발생하면 실행되지 않는다.

---

### `@AfterThrowing`

대상 메서드에서 예외가 발생했을 때 실행된다.

```
@AfterThrowing(
    pointcut = "execution(* com.example.service..*(..))",
    throwing = "exception"
)
public void afterThrowing(Exception exception) {
    System.out.println("예외 발생: " + exception.getMessage());
}
```

주로 다음과 같은 용도로 사용한다.

- 예외 로그 기록
- 모니터링 시스템에 오류 전송
- 장애 통계 수집

주의할 점은 예외를 무조건 여기서 처리해야 한다는 뜻이 아니라는 것이다. Advice에서 로그만 기록하고 예외는 원래 호출자에게 전달되게 할 수도 있다.

---

### `@After`

성공 또는 실패 여부와 관계없이 메서드 실행이 끝나면 동작한다.

```
@After("execution(* com.example.service..*(..))")
public void after() {
    System.out.println("메서드 실행 종료");
}
```

Java의 `finally`와 비슷한 성격이다.

---

### `@Around`

메서드 실행 전후를 모두 제어할 수 있다.

```
@Around("execution(* com.example.service..*(..))")
public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
    System.out.println("실행 전");

    Object result = joinPoint.proceed();

    System.out.println("실행 후");

    return result;
}
```

가장 강력하고 자주 사용되지만 주의해서 사용해야 한다.

`@Around` Advice에서 다음 코드를 호출하지 않으면 실제 대상 메서드가 실행되지 않는다.

```
joinPoint.proceed();
```

또한 원래 메서드의 반환값이 있다면 일반적으로 그 반환값을 돌려줘야 한다.

```
Object result = joinPoint.proceed();
return result;
```

실행 시간처럼 예외가 발생하더라도 후처리가 필요한 경우에는 `try-finally`를 사용한다.

```
@Around("@annotation(MeasureTime)")
public Object measure(ProceedingJoinPoint joinPoint) throws Throwable {
    long start = System.nanoTime();

    try {
        return joinPoint.proceed();
    } finally {
        long elapsed = System.nanoTime() - start;
        System.out.println("실행 시간: " + elapsed);
    }
}
```

---

# Pointcut

Pointcut은 Advice를 어떤 대상에 적용할지 결정하는 조건이다.

다음 표현식은 `service` 패키지와 하위 패키지에 있는 모든 메서드에 적용한다는 뜻이다.

```
execution(* com.example.service..*(..))
```

처음에는 문법이 복잡해 보이지만 다음 정도만 이해하면 충분하다.

```
execution(* com.example.service..*(..))
          │ └───────────┬──────────┘
          │             └─ 대상 패키지와 메서드
          └─ 모든 반환 타입
```

각 부분을 풀면 다음과 같다.

- 첫 번째 `*`: 모든 반환 타입
- `com.example.service..`: 해당 패키지와 모든 하위 패키지
- 두 번째 `*`: 모든 메서드 이름
- `(..)`: 모든 개수와 타입의 파라미터

### 특정 클래스의 모든 메서드

```
execution(* com.example.service.OrderService.*(..))
```

### 특정 메서드

```
execution(* com.example.service.OrderService.createOrder(..))
```

### 특정 어노테이션이 붙은 메서드

```
@annotation(com.example.aop.MeasureTime)
```

실무에서는 복잡한 `execution` 표현식보다 커스텀 어노테이션을 사용하는 방식이 읽기 쉬운 경우가 많다.

---

## 커스텀 어노테이션과 AOP

먼저 어노테이션을 정의한다.

```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MeasureTime {
}
```

Aspect에서 해당 어노테이션을 Pointcut으로 지정한다.

```
@Aspect
@Component
public class ExecutionTimeAspect {

    @Around("@annotation(MeasureTime)")
    public Object measure(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.nanoTime();

        try {
            return joinPoint.proceed();
        } finally {
            long elapsed = System.nanoTime() - start;

            System.out.println(
                joinPoint.getSignature().toShortString()
                    + ": "
                    + elapsed
                    + "ns"
            );
        }
    }
}
```

적용할 메서드에 어노테이션을 붙인다.

```
@Service
public class OrderService {

    @MeasureTime
    public void createOrder() {
        // 주문 생성
    }
}
```

이 방식의 장점은 적용 대상이 코드에 명확히 드러난다는 것이다.

---

# Spring AOP는 Proxy 기반이다

주니어 개발자가 Spring AOP에서 가장 중요하게 알아야 할 부분이다.

Spring은 일반적으로 실제 객체를 직접 주입하지 않고, 실제 객체를 감싼 Proxy를 주입한다.

```
@Service
public class OrderService {

    public void createOrder() {
        // 실제 비즈니스 로직
    }
}
```

Spring 내부에서는 개념적으로 다음과 비슷한 객체가 만들어진다.

```
public class OrderServiceProxy {

    private final OrderService target;

    public void createOrder() {
        System.out.println("실행 전");

        target.createOrder();

        System.out.println("실행 후");
    }
}
```

실제 Proxy 코드가 정확히 이렇게 생성되는 것은 아니지만, 원리를 이해하기 위한 모델로는 충분하다.

AOP가 동작하려면 호출이 Proxy를 통과해야 한다.

```
외부 객체 → Proxy → 실제 객체
```

Proxy를 거치지 않고 실제 객체 내부에서 직접 호출하면 AOP가 동작하지 않을 수 있다.

---

# 내부 호출 문제: self-invocation

Spring AOP에서 아주 자주 발생하는 문제이다.

```
@Service
public class OrderService {

    public void createOrder() {
        saveOrder();
    }

    @MeasureTime
    public void saveOrder() {
        // 저장 로직
    }
}
```

외부에서 다음과 같이 호출한다고 해 보면

```
orderService.createOrder();
```

`createOrder()`가 내부에서 `saveOrder()`를 호출한다.

```
saveOrder();
```

이 호출은 사실상 다음과 같다.

```
this.saveOrder();
```

즉, Proxy를 거치지 않고 현재 객체가 자신의 메서드를 직접 호출한다.

```
외부 → Proxy → createOrder()
                   ↓
             this.saveOrder()
```

따라서 `saveOrder()`에 설정한 AOP가 동작하지 않을 수 있다.

이 문제는 `@Transactional`에서도 매우 자주 나타난다.

```
public void createOrder() {
    saveOrder(); // 내부 호출
}

@Transactional
public void saveOrder() {
}
```

이 경우 `saveOrder()`의 트랜잭션이 기대한 방식으로 시작되지 않을 수 있다.

### 가장 일반적인 해결 방법

AOP가 적용될 메서드를 별도 Bean으로 분리한다.

```
@Service
public class OrderService {

    private final OrderWriter orderWriter;

    public OrderService(OrderWriter orderWriter) {
        this.orderWriter = orderWriter;
    }

    public void createOrder() {
        orderWriter.saveOrder();
    }
}
```

```
@Service
public class OrderWriter {

    @Transactional
    public void saveOrder() {
        // 저장 로직
    }
}
```

이제 호출이 다른 Bean의 Proxy를 통과하므로 AOP가 적용된다.

```
OrderService → OrderWriter Proxy → OrderWriter
```

주니어 단계에서는 다음 문장을 기억하면 충분하다.

> Spring AOP는 Proxy를 통과한 외부 호출에 적용되며, 같은 객체 내부의 메서드 호출에는 적용되지 않을 수 있다.

---

# Proxy의 종류

Spring에서는 대표적으로 두 가지 방식의 Proxy를 사용한다.

### JDK Dynamic Proxy

인터페이스를 기반으로 Proxy를 만든다.

```
public interface OrderService {
    void createOrder();
}
```

```
@Service
public class OrderServiceImpl implements OrderService {
    @Override
    public void createOrder() {
    }
}
```

### CGLIB Proxy

클래스를 상속받는 방식으로 Proxy를 만든다.

```
@Service
public class OrderService {
    public void createOrder() {
    }
}
```

주니어가 내부 구현 원리까지 깊게 알 필요는 없다. 다만 다음 정도는 알아두면 좋다.

- 인터페이스 기반 Proxy와 클래스 기반 Proxy가 존재한다.
- Proxy 방식에 따라 타입과 메서드 제약이 생길 수 있다.
- `final` 클래스나 `final` 메서드는 상속 기반 Proxy로 가로채기 어렵다.
- 문제가 생겼을 때 “주입된 객체가 실제 클래스가 아니라 Proxy일 수 있다”는 것을 떠올려야 한다.

---

# AOP의 대표적인 실제 사용 사례

## 로깅

```
@Around("execution(* com.example.service..*(..))")
public Object log(ProceedingJoinPoint joinPoint) throws Throwable {
    String method = joinPoint.getSignature().toShortString();

    log.info("시작: {}", method);

    try {
        Object result = joinPoint.proceed();
        log.info("성공: {}", method);
        return result;
    } catch (Exception e) {
        log.error("실패: {}", method, e);
        throw e;
    }
}
```

중요한 점은 예외를 기록한 뒤 원래 예외를 다시 던진다는 것이다.

```
throw e;
```

예외를 삼켜버리면 호출자는 작업이 실패했다는 사실을 모를 수 있다.

---

## 실행 시간 측정

```
@Around("@annotation(MeasureTime)")
public Object measureTime(ProceedingJoinPoint joinPoint) throws Throwable {
    long start = System.nanoTime();

    try {
        return joinPoint.proceed();
    } finally {
        long elapsed = System.nanoTime() - start;
        log.info(
            "{} 실행 시간: {}ms",
            joinPoint.getSignature().toShortString(),
            elapsed / 1_000_000
        );
    }
}
```

---

## 트랜잭션

```
@Transactional
public void createOrder() {
    orderRepository.save(order);
    paymentRepository.save(payment);
}
```

개념적으로는 Proxy가 다음과 비슷한 작업을 수행한다.

```
1. 트랜잭션 시작
2. createOrder() 실행
3. 성공하면 commit
4. 예외가 발생하면 rollback
```

실제 세부 동작은 더 복잡하지만 주니어 단계에서는 이 정도의 흐름을 이해하면 충분하다.

---

## 권한 확인

```
@CheckAdmin
public void deleteMember(Long memberId) {
    memberRepository.deleteById(memberId);
}
```

Aspect에서 어노테이션을 확인하여 관리자 권한을 검사할 수 있다.

단, 보안처럼 중요한 기능은 직접 만든 단순 AOP보다 Spring Security처럼 검증된 프레임워크를 사용하는 편이 일반적이다.

---

# `ProceedingJoinPoint`로 얻을 수 있는 정보

`@Around`에서는 `ProceedingJoinPoint`를 통해 호출 정보를 얻을 수 있다.

### 메서드 정보

```
joinPoint.getSignature().getName();
```

### 클래스 정보

```
joinPoint.getTarget().getClass();
```

### 전달된 파라미터

```
Object[] args = joinPoint.getArgs();
```

예:

```
@Around("@annotation(LogExecution)")
public Object log(ProceedingJoinPoint joinPoint) throws Throwable {
    String methodName = joinPoint.getSignature().getName();
    Object[] args = joinPoint.getArgs();

    log.info("메서드: {}, 인자: {}", methodName, Arrays.toString(args));

    return joinPoint.proceed();
}
```

다만 모든 파라미터를 무조건 로그로 남기면 안 된다.

다음 정보가 포함될 수 있기 때문이다.

- 비밀번호
- 액세스 토큰
- 주민등록번호
- 카드번호
- 개인정보
- 매우 큰 요청 데이터

AOP 로깅을 작성할 때는 민감 정보가 기록되지 않도록 주의해야 한다.

---

# AOP를 사용하기 좋은 경우

AOP는 다음 조건에 해당할 때 유용하다.

- 여러 클래스에 반복되는 기능이다.
- 핵심 비즈니스 로직과 직접적인 관련이 없다.
- 적용 규칙을 비교적 일관되게 정의할 수 있다.
- 메서드 실행 전후에 처리할 수 있다.

대표적으로 다음 기능이 적합하다.

- 실행 시간 측정
- 공통 로깅
- 트랜잭션
- 모니터링
- 감사 기록
- 공통 권한 검사

---

# AOP를 사용하지 않는 것이 좋은 경우

AOP가 코드 중복을 줄여준다고 해서 모든 중복 로직을 AOP로 처리하면 안 된다.

## 핵심 비즈니스 로직

다음과 같은 내용은 코드에 명시적으로 보여야 한다.

- 주문 금액 계산
- 할인 정책 적용
- 재고 차감
- 결제 승인
- 포인트 지급

예를 들어 “주문이 완료되면 포인트를 지급한다”는 것은 단순한 부가 기능이 아니라 중요한 비즈니스 규칙일 수 있다. 이를 AOP에 숨기면 코드를 읽는 사람이 전체 흐름을 파악하기 어려워진다.

```
public void completeOrder() {
    order.complete();
    pointService.giveOrderPoint(order);
}
```

이런 코드는 명시적으로 작성하는 편이 이해하기 쉽다.

## 특정 메서드 한 곳에서만 사용하는 기능

한 곳에서만 사용하는 짧은 로직이라면 AOP로 분리하는 것이 오히려 복잡도를 높일 수 있다.

## 복잡한 흐름 제어

Advice가 반환값을 바꾸거나, 예외를 삼키거나, 대상 메서드 실행 여부를 복잡하게 결정하면 코드의 실제 동작을 추적하기 어려워진다.

---

# AOP의 장점과 단점

## 장점

- 중복 코드를 줄일 수 있다.
- 핵심 로직과 부가 기능을 분리할 수 있다.
- 공통 정책을 한 곳에서 변경할 수 있다.
- 로깅이나 모니터링 적용을 일관되게 관리할 수 있다.
- 비즈니스 코드의 가독성이 좋아질 수 있다.

## 단점

- 코드만 보고 실제 실행 흐름을 알기 어려울 수 있다.
- Proxy와 내부 호출 문제로 예상과 다르게 동작할 수 있다.
- Pointcut 범위가 잘못되면 의도하지 않은 메서드에 적용될 수 있다.
- 여러 Aspect가 동시에 적용되면 실행 순서를 파악하기 어려울 수 있다.
- 과도하게 사용하면 중요한 로직이 숨겨진다.
- 리플렉션, Proxy, 로그 처리 등으로 추가 비용이 발생할 수 있다.

AOP의 목적은 코드를 무조건 짧게 만드는 것이 아니라 관심사를 분리하여 유지보수성을 높이는 것이다.

---

# 여러 Aspect가 적용될 때

한 메서드에 다음 기능이 동시에 적용될 수 있다.

- 권한 검사
- 트랜잭션
- 로깅
- 실행 시간 측정

이때 실행 순서가 중요해질 수 있다.

```
@Aspect
@Component
@Order(1)
public class LoggingAspect {
}
```

```
@Aspect
@Component
@Order(2)
public class ExecutionTimeAspect {
}
```

숫자가 작을수록 바깥쪽에서 먼저 시작하는 것으로 이해하면 된다.

```
Logging 시작
  ExecutionTime 시작
    실제 메서드 실행
  ExecutionTime 종료
Logging 종료
```

주니어 단계에서는 다음만 기억하면 충분하다.

- 한 메서드에 여러 Aspect가 적용될 수 있다.
- 실행 순서가 결과에 영향을 줄 수 있다.
- 필요하면 `@Order`로 순서를 지정할 수 있다.
- 순서에 과도하게 의존하는 설계는 복잡해질 수 있다.

---

# AOP 구현 시 자주 하는 실수

## `proceed()`를 호출하지 않음

```
@Around("@annotation(MeasureTime)")
public Object measure(ProceedingJoinPoint joinPoint) {
    log.info("실행");
    return null;
}
```

실제 대상 메서드가 실행되지 않는다.

올바른 기본 형태는 다음과 같다.

```
return joinPoint.proceed();
```

---

## 반환값을 무시함

```
@Around("@annotation(MeasureTime)")
public Object measure(ProceedingJoinPoint joinPoint) throws Throwable {
    joinPoint.proceed();
    return null;
}
```

대상 메서드는 실행되지만 원래 반환값이 사라진다.

```
Object result = joinPoint.proceed();
return result;
```

---

## 예외를 삼킴

```
try {
    return joinPoint.proceed();
} catch (Exception e) {
    log.error("오류", e);
    return null;
}
```

원래 실패해야 할 작업이 성공한 것처럼 보일 수 있다.

특별한 요구사항이 없다면 일반적으로 예외를 다시 던진다.

```
try {
    return joinPoint.proceed();
} catch (Exception e) {
    log.error("오류", e);
    throw e;
}
```

---

## 내부 호출에 AOP가 적용될 것으로 기대함

```
public void outer() {
    inner();
}

@Transactional
public void inner() {
}
```

`inner()` 호출이 Proxy를 통과하지 않으므로 기대한 AOP가 동작하지 않을 수 있다.

---

## Pointcut 범위를 지나치게 넓게 설정함

```
@Around("execution(* com.example..*(..))")
```

거의 모든 메서드에 적용될 수 있다. 불필요한 로그와 성능 저하가 발생할 수 있고, 예상하지 못한 곳에서 Advice가 동작할 수도 있다.

가능하면 적용 범위를 명확하게 제한해야 한다.

```
@Around("@annotation(MeasureTime)")
```

---

## 파라미터를 전부 로그로 기록함

```
log.info("args={}", Arrays.toString(joinPoint.getArgs()));
```

비밀번호나 개인정보가 로그에 남을 수 있다. 로깅 대상과 마스킹 정책을 먼저 정해야 한다.

---

# AOP 테스트

AOP는 Proxy를 통해 적용되기 때문에 Aspect 객체를 직접 생성해서 테스트하는 것만으로는 부족할 수 있다.

실제 Spring Context에서 Bean을 주입받아 호출하는 테스트가 도움이 된다.

```
@SpringBootTest
class ExecutionTimeAspectTest {

    @Autowired
    OrderService orderService;

    @Test
    void 실행_시간_AOP가_적용된다() {
        orderService.createOrder();
    }
}
```

가능하면 단순히 “호출해도 예외가 없다”만 검사하기보다 다음을 검증하는 것이 좋다.

- Advice가 실제로 호출되었는가
- 대상 메서드도 정상적으로 실행되었는가
- 반환값이 유지되는가
- 예외가 원래대로 전달되는가
- Pointcut 범위 밖의 메서드에는 적용되지 않는가