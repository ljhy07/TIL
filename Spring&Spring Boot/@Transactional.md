## `@Transactional`이란?

`@Transactional`은 여러 데이터베이스 작업을 **하나의 논리적 작업 단위**로 묶는 애너테이션이다.

```
@Service
public class OrderService {

    @Transactional
    public void placeOrder(OrderRequest request) {
        orderRepository.save(request.toOrder());
        paymentService.pay(request);
        inventoryService.decreaseStock(request.productId());
    }
}
```

위 메서드에서 결제나 재고 차감 중 예외가 발생하면, 앞서 저장한 주문까지 취소할 수 있다.

- 모두 성공하면 `COMMIT`
- 하나라도 실패하면 `ROLLBACK`

핵심 목적은 데이터가 일부만 변경되어 불일치 상태에 빠지는 것을 방지하는 것이다.

---

## 트랜잭션의 ACID 특성

데이터베이스 트랜잭션은 일반적으로 ACID를 보장하려고 한다.

### Atomicity — 원자성

작업 전체가 하나의 단위로 실행된다.

```
주문 저장 성공
결제 실패
→ 주문 저장도 롤백
```

### Consistency — 일관성

트랜잭션 전후에 데이터가 정의된 규칙을 만족해야 한다.

예를 들어 재고가 0보다 작아지면 안 된다는 규칙이 있다면, 트랜잭션 완료 후에도 그 규칙이 유지되어야 한다.

### Isolation — 격리성

동시에 실행되는 트랜잭션들이 서로의 중간 상태를 함부로 보지 못하도록 한다.

### Durability — 지속성

커밋된 데이터는 장애가 발생하더라도 보존되어야 한다.

---

## Spring 동작

일반적인 Spring 환경에서는 프록시가 `@Transactional` 메서드를 감싼다.

```
호출자
  ↓
Spring 트랜잭션 프록시
  1. 트랜잭션 시작
  2. 실제 메서드 호출
  3. 성공하면 커밋
  4. 예외면 롤백
  ↓
실제 서비스 객체
```

개념적으로는 다음 코드와 비슷하다.

```
transactionManager.begin();

try {
    target.placeOrder(request);
    transactionManager.commit();
} catch (RuntimeException e) {
    transactionManager.rollback();
    throw e;
}
```

개발자가 직접 `begin`, `commit`, `rollback`을 작성하지 않아도 Spring이 처리해 주는 것이다.

---

## 위치

### 메서드 적용

```
@Transactional
public void createOrder() {
    // ...
}
```

해당 메서드에만 적용된다.

### 클래스 적용

```
@Service
@Transactional
public class OrderService {

    public void createOrder() {
        // 트랜잭션 적용
    }

    public void cancelOrder() {
        // 트랜잭션 적용
    }
}
```

클래스의 대상 메서드 전체에 기본 설정이 적용된다.

메서드에 별도의 설정이 있으면 일반적으로 메서드 설정이 우선한다.

```
@Service
@Transactional(readOnly = true)
public class ProductService {

    public Product findProduct(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    @Transactional
    public void updateProduct(Product product) {
        productRepository.save(product);
    }
}
```

이 패턴은 조회가 많은 서비스에서 자주 사용된다.

---

## 기본 롤백 규칙

Spring의 기본 설정은 모든 예외를 똑같이 처리하지 않는다.

|예외 종류|기본 동작|
|---|---|
|`RuntimeException`|롤백|
|`Error`|롤백|
|체크 예외 `Exception`|커밋 가능|

예를 들어:

```
@Transactional
public void process() throws IOException {
    repository.save(entity);
    throw new IOException();
}
```

`IOException`은 체크 예외이므로 기본 설정에서는 롤백되지 않을 수 있다.

체크 예외도 롤백하려면 명시합니다.

```
@Transactional(rollbackFor = Exception.class)
public void process() throws Exception {
    repository.save(entity);
    throw new Exception();
}
```

특정 예외에 대해서는 롤백하지 않도록 설정할 수도 있다.

```
@Transactional(noRollbackFor = BusinessWarningException.class)
public void process() {
    // ...
}
```

### 예외를 잡아버리면 롤백되지 않는다

```
@Transactional
public void process() {
    try {
        repository.save(entity);
        externalService.call();
    } catch (RuntimeException e) {
        log.error("실패", e);
    }
}
```

예외가 메서드 밖으로 전달되지 않았으므로, 트랜잭션 프록시는 정상 종료로 판단해 커밋할 수 있다.

대개는 다시 던져야 한다.

```
@Transactional
public void process() {
    try {
        repository.save(entity);
        externalService.call();
    } catch (RuntimeException e) {
        log.error("실패", e);
        throw e;
    }
}
```

---

## 전파 속성 `propagation`

전파 속성은 이미 트랜잭션이 실행 중일 때 새로운 `@Transactional` 메서드를 호출하면 어떻게 할지를 결정한다.

```
@Transactional(propagation = Propagation.REQUIRED)
```

### `REQUIRED` — 기본값

기존 트랜잭션이 있으면 참여하고, 없으면 새로 생성한다.

```
OrderService 트랜잭션
 └─ PaymentService도 같은 트랜잭션에 참여
```

```
@Transactional
public void createOrder() {
    orderRepository.save(order);
    paymentService.pay();
}
```

`pay()`에서 런타임 예외가 발생하면 주문 저장도 함께 롤백된다.

### `REQUIRES_NEW`

기존 트랜잭션을 잠시 중단하고 별도의 새 트랜잭션을 시작한다.

```
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void writeAuditLog() {
    auditLogRepository.save(log);
}
```

외부 주문 트랜잭션이 롤백되더라도 감사 로그는 별도로 커밋할 수 있다.

```
외부 트랜잭션: 주문 처리 → 롤백
내부 트랜잭션: 감사 로그 → 커밋
```

다만 새 데이터베이스 커넥션이 필요할 수 있어 커넥션 풀 부족과 교착 상태 가능성을 고려해야 한다.

### `SUPPORTS`

기존 트랜잭션이 있으면 참여하고, 없으면 트랜잭션 없이 실행한다.

### `MANDATORY`

기존 트랜잭션이 반드시 있어야 합니다. 없으면 예외가 발생한다.

```
@Transactional(propagation = Propagation.MANDATORY)
public void decreaseStock() {
    // 상위 업무 트랜잭션 안에서만 실행하도록 강제
}
```

### `NOT_SUPPORTED`

기존 트랜잭션이 있어도 중단하고 트랜잭션 없이 실행한다.

### `NEVER`

트랜잭션이 존재하면 예외가 발생한다.

### `NESTED`

기존 트랜잭션 안에 저장 지점, 즉 savepoint를 생성한다. 내부 작업만 부분 롤백할 수 있지만, 트랜잭션 매니저와 데이터베이스의 지원 여부를 확인해야 한다.

---

## 격리 수준 `isolation`

격리 수준은 여러 트랜잭션이 동시에 같은 데이터를 읽고 수정할 때 어느 정도까지 서로 격리할지 결정한다.

```
@Transactional(isolation = Isolation.READ_COMMITTED)
```

### 동시성 문제

- Dirty Read: 다른 트랜잭션이 아직 커밋하지 않은 데이터를 읽음
- Non-repeatable Read: 같은 행을 두 번 읽었는데 값이 달라짐
- Phantom Read: 같은 조건으로 조회했는데 행의 개수가 달라짐

### 격리 수준

|격리 수준|특징|
|---|---|
|`DEFAULT`|데이터베이스 기본값 사용|
|`READ_UNCOMMITTED`|커밋되지 않은 데이터도 읽을 수 있음|
|`READ_COMMITTED`|커밋된 데이터만 읽음|
|`REPEATABLE_READ`|같은 행을 반복 조회할 때 일관성을 강화|
|`SERIALIZABLE`|가장 강한 격리, 동시 처리 성능은 낮아질 수 있음|

격리 수준을 높이면 정합성은 강해지는 경향이 있지만, 잠금 대기와 교착 상태가 늘고 처리량이 낮아질 수 있다.

또한 실제 동작은 데이터베이스의 MVCC와 잠금 구현에 따라 달라진다. 단순히 격리 수준만 높인다고 모든 동시성 문제가 자동으로 해결되지는 않는다.

---

## `readOnly = true`

조회 전용 트랜잭션을 표현한다.

```
@Transactional(readOnly = true)
public Product findProduct(Long id) {
    return productRepository.findById(id).orElseThrow();
}
```

장점은 환경에 따라 다음과 같다.

- JPA/Hibernate의 불필요한 변경 감지 비용 감소
- flush 동작 최적화
- 데이터베이스 또는 라우팅 설정에 조회 전용 힌트 제공
- 코드의 의도를 명확하게 표현

하지만 이것을 절대적인 쓰기 방지 장치로 보면 안 된다.

```
@Transactional(readOnly = true)
public void updateProduct() {
    // 환경에 따라 변경이 무시되거나,
    // SQL이 실행되거나, 예외가 발생할 수 있음
}
```

실제 쓰기 차단 여부는 JPA 구현체, JDBC 드라이버, 데이터베이스 설정에 따라 달라질 수 있다.

---

## `timeout`

트랜잭션의 제한 시간을 설정할 수 있다.

```
@Transactional(timeout = 5)
public void process() {
    // 제한 시간: 5초
}
```

장시간 실행되는 트랜잭션은 다음 문제를 만들 수 있다.

- 커넥션 장기 점유
- 잠금 대기 증가
- 교착 상태 가능성 증가
- 전체 처리량 감소

다만 제한 시간이 정확히 어떤 작업에 적용되는지는 트랜잭션 매니저와 데이터베이스 드라이버에 따라 차이가 있다.

---

## 자기 자신 호출

Spring의 기본 프록시 방식에서는 같은 객체 내부에서 메서드를 호출하면 프록시를 거치지 않다.

```
@Service
public class OrderService {

    public void createOrder() {
        saveOrder(); // 자기 자신 호출
    }

    @Transactional
    public void saveOrder() {
        // 트랜잭션이 적용되지 않을 수 있음
    }
}
```

실제 호출 흐름이 다음과 같기 때문이다.

```
외부 → 프록시 → createOrder()
                   └─ this.saveOrder()
```

`saveOrder()` 호출은 프록시를 다시 통과하지 않는다.

일반적으로 트랜잭션 메서드를 별도의 Spring Bean으로 분리한다.

```
@Service
public class OrderService {

    private final OrderWriter orderWriter;

    public void createOrder() {
        orderWriter.saveOrder();
    }
}

@Service
public class OrderWriter {

    @Transactional
    public void saveOrder() {
        // 트랜잭션 적용
    }
}
```

---

## `private` 메서드의 문제

기본적인 프록시 기반 사용에서는 외부 호출이 가능한 메서드에 `@Transactional`을 붙이는 것이 안전하다.

```
@Transactional
private void saveOrder() {
    // 일반적인 프록시 방식에서는 기대대로 적용되지 않음
}
```

Spring 버전과 프록시 종류에 따라 세부 지원 범위가 다를 수 있지만, 애플리케이션 설계에서는 보통 `public` 서비스 메서드를 트랜잭션 경계로 사용한다.

---

## JPA 변경 감지와의 관계

JPA에서는 트랜잭션 안에서 조회한 엔티티가 영속 상태가 된다.

```
@Transactional
public void changeName(Long memberId, String name) {
    Member member = memberRepository.findById(memberId)
        .orElseThrow();

    member.changeName(name);
}
```

명시적인 `save()`가 없어도 트랜잭션 커밋 시점에 변경 사항이 감지되어 `UPDATE`가 실행될 수 있다.

```
트랜잭션 시작
→ 엔티티 조회
→ 객체 상태 변경
→ flush
→ UPDATE 실행
→ commit
```

반대로 트랜잭션 밖에서는 엔티티가 분리 상태가 되어 변경 감지가 작동하지 않을 수 있다.

---

## `flush`와 `commit`은 다르다

- `flush`: 영속성 컨텍스트의 변경 내용을 SQL로 데이터베이스에 전달
- `commit`: 트랜잭션을 최종 확정

```
@Transactional
public void create() {
    repository.save(entity);
    repository.flush();

    throw new RuntimeException();
}
```

`flush()`로 `INSERT` SQL이 실행됐더라도 이후 롤백되면 데이터는 최종 저장되지 않는다.

또한 제약조건 위반 같은 오류는 `save()` 호출이 아니라 `flush` 또는 커밋 시점에 발생할 수 있다.

---

## 트랜잭션 범위는 짧게 유지하는 것이 좋다

다음처럼 외부 API 호출을 데이터베이스 트랜잭션 안에 포함하면 위험할 수 있다.

```
@Transactional
public void processOrder() {
    orderRepository.save(order);

    paymentApi.call(); // 오래 걸릴 수 있음

    inventoryRepository.decreaseStock();
}
```

외부 API가 느려지는 동안:

- DB 커넥션을 계속 점유하고
- 잠금을 오래 유지하며
- 타임아웃과 장애 가능성을 높입니다.

그렇다고 외부 호출을 무조건 트랜잭션 밖으로 빼면 DB 작업과 외부 시스템 작업의 원자성이 깨진다. 이런 분산 트랜잭션 문제에는 상황에 따라 다음 패턴을 고려한다.

- Transactional Outbox
- Saga
- 이벤트 기반 처리
- 재시도와 멱등성
- 보상 트랜잭션

`@Transactional`은 일반적으로 하나의 데이터베이스 내부 정합성을 관리하며, 외부 API·메시지 브로커·다른 데이터베이스까지 자동으로 원자적으로 묶어주지는 않는다.

---

## 흔히 권장되는 사용 위치

보통 Repository보다 Service 계층에서 업무 단위로 설정한다.

```
@Service
@RequiredArgsConstructor
public class TransferService {

    private final AccountRepository accountRepository;

    @Transactional
    public void transfer(
        Long fromId,
        Long toId,
        BigDecimal amount
    ) {
        Account from = accountRepository.findById(fromId)
            .orElseThrow();

        Account to = accountRepository.findById(toId)
            .orElseThrow();

        from.withdraw(amount);
        to.deposit(amount);
    }
}
```

트랜잭션의 경계가 “한 번의 송금”이라는 비즈니스 작업과 일치한다.

Repository 메서드 각각에 별도 트랜잭션만 적용하면 출금은 성공하고 입금은 실패하는 부분 완료 문제가 생길 수 있다.