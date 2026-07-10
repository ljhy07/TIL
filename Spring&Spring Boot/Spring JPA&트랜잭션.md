## Transaction

Spring에서 `@Transactional`은 “이 메서드를 하나의 트랜잭션 경계로 실행하라”는 선언이다. 보통 서비스 계층 메서드에 붙이고, Spring AOP 프록시가 메서드 호출 전후로 트랜잭션을 시작, 커밋, 롤백한다.

```
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Long userId, Long itemId) {
        Order order = orderRepository.save(...);
        paymentRepository.save(...);
        stock.decrease();
        // 정상 종료 -> commit
        // RuntimeException/Error 발생 -> rollback
    }
}
```

트랜잭션 경계는 보통 다음을 의미한다.

1. 메서드 진입 전 트랜잭션 시작 또는 기존 트랜잭션 참여
2. 같은 트랜잭션 안에서 JPA `EntityManager`와 영속성 컨텍스트 사용
3. 엔티티 변경은 dirty checking 대상
4. 정상 종료 시 flush 후 commit
5. 예외 발생 시 rollback 또는 rollback-only 처리

## JPA에서 중요한 점

JPA는 트랜잭션 안에서 영속성 컨텍스트를 통해 엔티티를 관리한다.

```
@Transactional
public void changeName(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    user.changeName("kim");
    // save()를 명시하지 않아도, 영속 상태 엔티티라면 commit 시 dirty checking으로 update
}
```

트랜잭션이 끝나면 엔티티는 보통 detached 상태가 된다. 그래서 트랜잭션 밖에서 lazy 로딩을 건드리면 `LazyInitializationException`이 날 수 있다. `OpenEntityManagerInView`가 켜져 있으면 조회는 되는 것처럼 보일 수 있지만, 서비스 트랜잭션 경계가 늘어난 것은 아니다.

### 기본 전파 : REQUIRED

`@Transactional`의 기본 전파는 `Propagation.REQUIRED`이다.

```
@Transactional
public void outer() {
    inner();
}

@Transactional(propagation = Propagation.REQUIRED)
public void inner() {
}
```

의미는 간단하다.

- 이미 트랜잭션이 있으면 그 트랜잭션에 참여
- 없으면 새 트랜잭션 생성

즉 `outer()`와 `inner()`는 같은 물리 트랜잭션을 쓴다. `inner()`에서 문제가 생겨 rollback-only로 표시되면, `outer()`에서 예외를 잡아도 최종 커밋 시점에 롤백될 수 있다. 이때 Spring은 바깥 코드가 “커밋된 줄 착각하지 않게” `UnexpectedRollbackException`을 던질 수 있다.

## 옵션 정리

|전파 옵션|기존 트랜잭션이 있으면|없으면|주 용도|
|---|---|---|---|
|`REQUIRED`|참여|새로 시작|기본값, 대부분의 서비스 로직|
|`REQUIRES_NEW`|기존 트랜잭션 suspend, 새 트랜잭션 시작|새로 시작|감사 로그, 독립 저장|
|`NESTED`|savepoint 생성|새로 시작|부분 롤백, JDBC savepoint 기반|
|`SUPPORTS`|참여|트랜잭션 없이 실행|읽기 보조 로직|
|`MANDATORY`|참여|예외|반드시 상위 트랜잭션 필요|
|`NOT_SUPPORTED`|기존 트랜잭션 suspend|트랜잭션 없이 실행|트랜잭션 밖에서 긴 작업|
|`NEVER`|예외|트랜잭션 없이 실행|트랜잭션이 있으면 안 되는 작업|

## REQUIRES_NEW

`REQUIRES_NEW`는 바깥 트랜잭션과 독립된 새 트랜잭션을 연다.

```
@Transactional
public void order() {
    try {
        payment();
    } catch (Exception e) {
        auditService.saveFailureLog(); // REQUIRES_NEW
        throw e;
    }
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveFailureLog() {
    auditRepository.save(...);
}
```

`order()`가 나중에 롤백되어도 `saveFailureLog()`는 이미 커밋될 수 있다. 다만 기존 트랜잭션을 붙잡은 채 새 DB 커넥션을 더 요구할 수 있으므로 커넥션 풀 부족에 주의해야 한다.

## NESTED

`NESTED`는 독립 트랜잭션이라기보다 같은 물리 트랜잭션 안에서 savepoint를 잡는 방식이다.

```
@Transactional
public void outer() {
    step1();

    try {
        nestedStep();
    } catch (Exception ignored) {
        // nestedStep만 savepoint로 롤백하고 outer는 계속 가능
    }

    step2();
}
```

JPA 환경에서는 `NESTED` 지원이 트랜잭션 매니저와 JDBC savepoint 지원에 의존한다. 순수 JPA 서비스에서는 보통 `REQUIRED`와 `REQUIRES_NEW`를 더 자주 쓴다.

## 롤백 규칙

기본 롤백 규칙은 중요하다.

```
@Transactional
public void run() {
    throw new RuntimeException(); // rollback
}
```

기본적으로 rollback 되는 예외:

- `RuntimeException`
- `Error`

기본적으로 rollback 되지 않는 예외:

- checked exception, 예: `IOException`, 직접 만든 `Exception`

checked exception도 롤백하려면 이렇게 지정해야한다.

```
@Transactional(rollbackFor = MyCheckedException.class)
public void run() throws MyCheckedException {
    throw new MyCheckedException();
}
```

반대로 특정 예외는 롤백하지 않게 할 수도 있다.

```
@Transactional(noRollbackFor = BusinessException.class)
```

주의할 점은 예외를 잡고 삼키면 Spring 입장에서는 정상 종료로 보일 수 있다는 것이다.

```
@Transactional
public void run() {
    try {
        risky();
    } catch (Exception e) {
        // 여기서 끝나면 commit될 수 있음
    }
}
```

이 경우 롤백하려면 다시 던지거나, 정말 필요할 때만 `setRollbackOnly()`를 사용한다.

## readOnly, isolation, timeout

```
@Transactional(
    readOnly = true,
    isolation = Isolation.READ_COMMITTED,
    timeout = 5
)
public List<User> findUsers() {
    return userRepository.findAll();
}
```

`readOnly = true`는 “읽기 전용 힌트”이다. Hibernate flush 최적화 등에 도움을 줄 수 있지만, 모든 DB에서 쓰기를 완벽히 막는 안전장치라고 보면 안 된다.

`isolation`은 동시성 격리 수준이다.

- `READ_UNCOMMITTED`: dirty read 가능
- `READ_COMMITTED`: dirty read 방지
- `REPEATABLE_READ`: non-repeatable read 방지
- `SERIALIZABLE`: 가장 강하지만 비용 큼
- `DEFAULT`: DB 기본값 사용

중요한 점은 이미 존재하는 트랜잭션에 `REQUIRED`로 참여하면 내부 메서드의 isolation, timeout, readOnly 설정이 기대처럼 새로 적용되지 않을 수 있다는 것이다. 이런 속성은 “새 트랜잭션을 시작할 때” 의미가 가장 크다.

## 프록시 함정

가장 많이 터지는 문제는 self-invocation이다.

```
@Service
public class UserService {
    public void outer() {
        inner(); // 같은 객체 내부 호출
    }

    @Transactional
    public void inner() {
    }
}
```

이 경우 `inner()` 호출이 Spring 프록시를 거치지 않기 때문에 `@Transactional`이 적용되지 않을 수 있다. 보통 해결은 트랜잭션 메서드를 다른 Spring Bean으로 분리하거나, 외부에서 프록시를 통해 호출되게 구조를 바꾸는 것이다.

또한 일반적으로 `private` 메서드, `final` 메서드, 직접 `new`로 만든 객체에는 기대한 방식으로 적용되지 않다.

## 사용 예시

대부분의 경우 트랜잭션은 repository가 아니라 service/use-case 단위에 둔다.

```
@Transactional
public void registerUser(...) {
    userRepository.save(user);
    profileRepository.save(profile);
    welcomeCouponRepository.save(coupon);
}
```

이렇게 해야 여러 repository 호출이 하나의 원자적 작업이 된다. Spring Data JPA repository 메서드에도 기본 트랜잭션 설정이 있지만, 여러 저장소 호출을 묶는 비즈니스 경계는 서비스 계층에서 잡는 것이 일반적이다.