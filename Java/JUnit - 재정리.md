# JUnit이 무엇인가

JUnit은 Java 코드가 예상대로 동작하는지 자동으로 검증하는 테스트 프레임워크이다.

```
@Test
void 두_수를_더한다() {
    Calculator calculator = new Calculator();

    int result = calculator.add(1, 2);

    assertEquals(3, result);
}
```

이 테스트가 검증하는 것은 다음 한 문장이다.

> `Calculator.add(1, 2)`의 결과는 `3`이어야 한다.

테스트를 작성하는 목적은 단순히 버그를 찾는 것만이 아니다.

- 기존 기능이 변경으로 인해 망가지지 않았는지 확인한다.
- 코드가 어떤 동작을 해야 하는지 문서처럼 보여준다.
- 리팩터링할 때 안전망을 제공한다.
- 사람이 반복해서 확인하던 작업을 자동화한다.

## 단위 테스트

일반적으로 클래스나 메서드처럼 작은 단위의 동작을 검증하는 테스트이다.

좋은 단위 테스트는 대체로 다음 성질을 가진다.

- 빠르게 실행된다.
- 실행 순서와 관계없이 독립적으로 동작한다.
- 같은 조건에서는 항상 같은 결과가 나온다.
- 실패했을 때 무엇이 잘못됐는지 쉽게 알 수 있다.
- 외부 DB, 네트워크, 실제 시간 등에 최대한 의존하지 않는다.

---

# JUnit 4와 JUnit 5 구분

프로젝트에서 가장 먼저 확인할 것은 import이다.

```
// JUnit 5
import org.junit.jupiter.api.Test;
```

```
// JUnit 4
import org.junit.Test;
```

새로운 프로젝트라면 보통 JUnit 5의 `org.junit.jupiter` 패키지를 사용한다. 검색해서 코드를 복사할 때 JUnit 4 예제와 JUnit 5 예제를 섞지 않도록 주의해야 한다.

JUnit 5 관련 용어

- **JUnit Platform**: 테스트를 찾아서 실행하는 기반
- **JUnit Jupiter**: JUnit 5 테스트를 작성하는 API와 실행 엔진
- **JUnit Vintage**: 예전 JUnit 3·4 테스트를 실행하기 위한 호환 기능

---

# 테스트 구조

```
@Test
void 상품을_추가하면_전체_금액이_증가한다() {
    // given: 테스트를 위한 준비
    Cart cart = new Cart();
    Product product = new Product("키보드", 50_000);

    // when: 검증할 동작 실행
    cart.add(product);

    // then: 결과 확인
    assertEquals(50_000, cart.getTotalPrice());
}
```

보통 아래로 구조를 많이 사용한다.

- Arrange → Act → Assert
- Given → When → Then

중요한 것은 주석 자체가 아니라 테스트에서 **준비, 실행, 검증이 구분되는 것**이다.

## 테스트 메서드 이름

다음과 같이 동작과 결과가 드러나는 이름이 좋다.

```
@Test
void add_returnsSumOfTwoNumbers() {
}
```

```
@Test
void 잔액보다_큰_금액을_출금하면_예외가_발생한다() {
}
```

팀의 컨벤션에 따라 영어 또는 한글을 사용할 수 있다. 이름만 보고 무엇을 검증하는지 알 수 있어야 한다.

---

# `@Test`와 기본 규칙

```
class CalculatorTest {

    @Test
    void add_returnsSum() {
        // 테스트 코드
    }
}
```

JUnit 5에서는 테스트 클래스와 메서드에 `public`을 붙이지 않아도 된다.

```
class CalculatorTest {       // 가능

    @Test
    void testSomething() {   // 가능
    }
}
```

테스트 메서드는 반환값을 사용하지 않으므로 보통 `void`이다.

다음처럼 테스트가 실행만 되고 검증이 없는 경우를 주의해야 한다.

```
@Test
void 잘못된_테스트() {
    calculator.add(1, 2);
}
```

예외가 발생하지 않는다는 사실만 확인하려는 특별한 경우가 아니라면, 테스트에는 결과를 확인하는 assertion이 있어야 한다.

---

# 반드시 알아야 하는 Assertion

Assertion은 실제 결과가 예상 결과와 일치하는지 확인하는 기능이다.

```
import static org.junit.jupiter.api.Assertions.*;
```

정적 import를 사용하면 `Assertions.assertEquals()` 대신 `assertEquals()`로 작성할 수 있다.

## `assertEquals()`

두 값이 같은지 확인한다.

```
assertEquals(3, calculator.add(1, 2));
```

인수 순서는 다음과 같다.

```
assertEquals(expected, actual);
```

즉, **예상값이 먼저이고 실제값이 나중**이다.

```
assertEquals(3, result, "덧셈 결과가 올바르지 않습니다.");
```

메시지는 테스트가 실패했을 때만 사용된다. 모든 assertion에 메시지를 억지로 작성할 필요는 없다.

### 객체 비교 시 주의

객체를 `assertEquals()`로 비교하면 일반적으로 객체의 `equals()`를 사용한다.

```
assertEquals(expectedMember, actualMember);
```

따라서 도메인 객체에 적절한 `equals()`가 구현되어 있지 않으면, 필드가 같아 보여도 테스트가 실패할 수 있다.

## `assertNotEquals()`

두 값이 달라야 하는 경우이다.

```
assertNotEquals(0, order.getId());
```

## `assertTrue()`, `assertFalse()`

조건식이 참 또는 거짓인지 확인한다.

```
assertTrue(member.isActive());
assertFalse(cart.isEmpty());
```

다만 구체적인 값 비교가 가능하다면 `assertEquals()`가 실패 원인을 더 명확하게 보여줄 수 있다.

```
// 상대적으로 모호함
assertTrue(result == 10);

// 실패 결과가 좀 더 명확함
assertEquals(10, result);
```

## `assertNull()`, `assertNotNull()`

```
assertNull(member.getDeletedAt());
assertNotNull(savedMember.getId());
```

## `assertSame()`과 `assertEquals()`의 차이

```
assertEquals(a, b); // 값이 같은지, 일반적으로 equals()로 비교
assertSame(a, b);   // 정확히 동일한 객체를 가리키는지 비교
```

```
String a = new String("hello");
String b = new String("hello");

assertEquals(a, b);    // 성공
assertSame(a, b);      // 실패
assertNotSame(a, b);   // 성공
```

`assertEquals()`가 훨씬 자주 사용된다.

## 배열 비교

배열은 `assertArrayEquals()`를 사용한다.

```
int[] expected = {1, 2, 3};
int[] actual = calculator.range(1, 3);

assertArrayEquals(expected, actual);
```

## 실수 비교

부동소수점은 오차가 발생할 수 있으므로 허용 오차를 지정한다.

```
assertEquals(0.3, calculator.add(0.1, 0.2), 0.000001);
```

## 여러 결과 함께 검증하기: `assertAll()`

```
assertAll(
    () -> assertEquals("민수", member.getName()),
    () -> assertEquals(20, member.getAge()),
    () -> assertTrue(member.isActive())
);
```

일반 assertion을 연속으로 작성하면 첫 번째 실패에서 테스트가 끝난다. `assertAll()`은 여러 검증을 실행한 뒤 실패 결과를 모아서 보여준다.

한 객체의 여러 중요한 속성을 함께 검증할 때 유용하지만, 서로 다른 동작을 한 테스트에 억지로 모으는 용도로 사용해서는 안 된다.

---

# 예외 테스트

## `assertThrows()`

특정 동작에서 예상한 예외가 발생하는지 확인한다.

```
@Test
void 0으로_나누면_예외가_발생한다() {
    IllegalArgumentException exception = assertThrows(
        IllegalArgumentException.class,
        () -> calculator.divide(10, 0)
    );

    assertEquals("0으로 나눌 수 없습니다.", exception.getMessage());
}
```

`assertThrows()`는 발생한 예외 객체를 반환하므로 메시지나 속성도 검증할 수 있다.

### 잘못된 예

```
@Test
void 잘못된_예외_테스트() {
    assertThrows(
        IllegalArgumentException.class,
        () -> {
            Calculator calculator = createCalculator();
            calculator.divide(10, 0);
            saveResult();
        }
    );
}
```

람다 안에 여러 동작을 넣으면 어느 코드가 예외를 발생시켰는지 모호해진다. 가능하면 실제로 예외가 발생해야 하는 동작만 넣는다.

```
assertThrows(
    IllegalArgumentException.class,
    () -> calculator.divide(10, 0)
);
```

## 직접 `try-catch`하지 않기

다음 방식은 피하는 것이 좋다.

```
@Test
void 예외를_검증한다() {
    try {
        calculator.divide(10, 0);
    } catch (IllegalArgumentException e) {
        assertEquals("0으로 나눌 수 없습니다.", e.getMessage());
    }
}
```

예외가 발생하지 않아도 테스트가 성공할 가능성이 있다. `assertThrows()`를 사용하는 것이 안전하다.

## `assertDoesNotThrow()`

예외가 발생하지 않아야 한다는 사실을 명시적으로 검증할 수 있다.

```
assertDoesNotThrow(() -> member.changeName("민수"));
```

다만 일반적으로 예상치 못한 예외가 발생하면 테스트는 원래 실패한다. 따라서 특별히 의도를 강조할 때만 사용해도 충분하다.

---

# 테스트 준비와 정리: 생명주기 Annotation

## `@BeforeEach`

각 테스트가 실행되기 전에 실행된다.

```
class CalculatorTest {

    private Calculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new Calculator();
    }

    @Test
    void 더하기() {
        assertEquals(3, calculator.add(1, 2));
    }

    @Test
    void 빼기() {
        assertEquals(1, calculator.subtract(3, 2));
    }
}
```

중복되는 준비 코드를 줄일 때 사용한다.

하지만 모든 것을 `@BeforeEach`로 숨기면 테스트가 읽기 어려워진다. 여러 테스트가 공통으로 필요로 하는 기본 객체 정도만 준비하는 것이 좋다. 특정 테스트에서만 필요한 값은 해당 테스트 안에서 직접 준비하는 편이 명확하다.

## `@AfterEach`

각 테스트 실행 후 호출된다.

```
@AfterEach
void tearDown() {
    resource.close();
}
```

파일, 서버, 외부 연결 등 명시적으로 정리해야 하는 자원이 있을 때 사용한다. 단순 객체는 보통 직접 정리할 필요가 없다.

## `@BeforeAll`, `@AfterAll`

클래스의 모든 테스트 전후에 한 번씩만 실행된다.

```
@BeforeAll
static void beforeAll() {
}

@AfterAll
static void afterAll() {
}
```

JUnit의 기본 설정에서는 `static`이어야 한다.

변경 가능한 객체를 공유하면 테스트 간 의존성이 생기기 쉽다.

## 기본적으로 테스트마다 새 인스턴스가 생성된다

JUnit Jupiter의 기본 동작에서는 테스트 메서드마다 테스트 클래스의 새 인스턴스가 만들어진다. 다음과 같은 테스트는 잘못된 가정을 하고 있다.

```
class CounterTest {

    private int count = 0;

    @Test
    void first() {
        count++;
        assertEquals(1, count);
    }

    @Test
    void second() {
        // first()에서 증가한 값이 전달된다고 생각하면 안 됨
        assertEquals(1, count);
    }
}
```

각 테스트는 다른 테스트가 실행되지 않아도 독립적으로 성공해야 한다.

---

# 테스트의 독립성은 매우 중요하다

좋은 테스트는 다음 조건을 만족해야 한다.

```
이 테스트 하나만 실행해도 성공한다.
모든 테스트를 함께 실행해도 성공한다.
실행 순서가 달라져도 성공한다.
여러 번 실행해도 같은 결과가 나온다.
```

다음과 같은 테스트는 피해야 한다.

```
@Test
void 회원을_먼저_등록한다() {
    sharedMemberId = memberService.register(...);
}

@Test
void 등록한_회원을_조회한다() {
    Member member = memberService.find(sharedMemberId);
}
```

두 번째 테스트가 첫 번째 테스트의 실행 결과에 의존한다. 각 테스트가 필요한 데이터를 직접 준비해야 한다.

`@Order`로 실행 순서를 강제해서 문제를 감추기보다는 테스트를 독립적으로 만드는 것이 우선이다.

---

# Parameterized Test

입력만 다르고 검증하는 동작이 같다면 테스트를 반복해서 작성하지 않고 매개변수 테스트를 사용할 수 있다.

## `@ValueSource`

한 개의 간단한 값을 여러 번 전달한다.

```
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

class PasswordValidatorTest {

    @ParameterizedTest
    @ValueSource(strings = {"12345678", "abcdefgh", "abcd1234"})
    void 여덟_글자_이상의_문자열은_길이_조건을_만족한다(String password) {
        assertTrue(password.length() >= 8);
    }
}
```

입력값마다 테스트가 한 번씩 실행된다.

## `@CsvSource`

여러 개의 인수를 전달할 때 사용한다.

```
@ParameterizedTest
@CsvSource({
    "1, 2, 3",
    "5, 7, 12",
    "-1, 1, 0"
})
void 두_수를_더한다(int left, int right, int expected) {
    assertEquals(expected, calculator.add(left, right));
}
```

## null과 빈 문자열

```
import org.junit.jupiter.params.provider.NullAndEmptySource;

@ParameterizedTest
@NullAndEmptySource
void 이름이_null이거나_빈_문자열이면_유효하지_않다(String name) {
    assertFalse(NameValidator.isValid(name));
}
```

## `@MethodSource`

복잡한 객체나 다양한 인수를 제공할 때 사용할 수 있다.

```
@ParameterizedTest
@MethodSource("invalidMembers")
void 유효하지_않은_회원은_생성할_수_없다(String name, int age) {
    assertThrows(
        IllegalArgumentException.class,
        () -> new Member(name, age)
    );
}

static Stream<Arguments> invalidMembers() {
    return Stream.of(
        Arguments.of("", 20),
        Arguments.of("민수", -1),
        Arguments.of(null, 30)
    );
}
```

`@ValueSource`와 `@CsvSource`를 우선 익히고, 필요할 때 `@MethodSource`를 찾아 사용할 수 있으면 충분하다.

---

# 있으면 좋은 Annotation

이 내용들은 필수까지는 아니지만 알고 있으면 테스트를 읽고 작성하는 데 도움이 된다.

## `@DisplayName`

IDE나 테스트 보고서에 읽기 쉬운 이름을 표시한다.

```
@Test
@DisplayName("잔액보다 큰 금액을 출금하면 예외가 발생한다")
void withdrawWithInsufficientBalance() {
}
```

메서드 이름 자체가 이미 충분히 명확하다면 반드시 사용할 필요는 없다.

## `@Nested`

관련 테스트를 상황별로 묶을 수 있다.

```
class StackTest {

    @Nested
    class 비어있는_스택 {

        @Test
        void pop하면_예외가_발생한다() {
        }
    }

    @Nested
    class 값이_들어있는_스택 {

        @Test
        void pop하면_마지막_값을_반환한다() {
        }
    }
}
```

상태나 조건에 따라 테스트를 그룹화할 때 좋다. 단순한 테스트까지 지나치게 중첩할 필요는 없다.

## `@Disabled`

테스트 실행을 임시로 막는다.

```
@Test
@Disabled("결제 서버 테스트 환경 복구 후 활성화")
void 결제한다() {
}
```

반드시 이유를 적고 최대한 빨리 해결해야 한다. 실패하는 테스트를 감추는 용도로 사용하면 안 된다.

## `@Tag`

테스트를 그룹으로 분류한다.

```
@Test
@Tag("slow")
void 오래_걸리는_테스트() {
}
```

CI에서 빠른 테스트와 느린 테스트를 분리할 때 사용한다.

## Assumption

특정 환경에서만 테스트를 실행해야 할 때 사용한다.

```
@Test
void 개발환경에서만_실행한다() {
    assumeTrue("dev".equals(System.getenv("PROFILE")));

    // 테스트
}
```

조건이 거짓이면 실패가 아니라 중단된 테스트로 처리된다. 일반적인 비즈니스 조건 분기에 사용하는 기능은 아니다.

## `@TempDir`

파일 테스트에서 임시 디렉터리를 안전하게 사용할 수 있다.

```
@TempDir
Path tempDir;

@Test
void 파일을_저장한다() throws IOException {
    Path file = tempDir.resolve("result.txt");

    Files.writeString(file, "hello");

    assertEquals("hello", Files.readString(file));
}
```

직접 프로젝트 폴더에 테스트 파일을 만들고 삭제하는 것보다 안전하다.

---

# 무엇을 테스트해야 하는가

## 공개된 동작을 테스트한다

다음처럼 private 메서드를 직접 테스트하려고 하지 않는 것이 좋다.

```
private int calculateDiscount() {
}
```

private 메서드는 구현 세부사항이다. 이를 사용하는 공개 메서드의 결과를 통해 검증한다.

```
@Test
void VIP_회원은_10퍼센트_할인을_받는다() {
    Order order = new Order(...);

    int price = order.calculateFinalPrice(VIP);

    assertEquals(90_000, price);
}
```

private 메서드를 직접 테스트해야만 할 정도로 로직이 복잡하다면, 별도의 책임을 가진 클래스로 분리할 수 있는지 고민하는 편이 좋다.

## 정상 상황만 테스트하지 않는다

적어도 다음 세 종류를 생각해 보는 습관이 좋다.

- 정상 입력
- 경계값
- 잘못된 입력과 예외

예를 들어 나이 제한이 `18세 이상`이라면 다음 값들이 중요하다.

```
17: 실패
18: 성공
19: 성공
```

빈 문자열을 허용하지 않는다면 다음도 생각할 수 있다.

```
null
""
" "
정상 문자열
```

모든 조합을 무작정 테스트하라는 뜻은 아니다. 동작이 달라지는 경계와 의미 있는 경우를 선택해야 한다.

---

# 단위 테스트와 통합 테스트 구분

순수 JUnit 단위 테스트는 보통 다음처럼 프레임워크를 실행하지 않는다.

```
class PriceCalculatorTest {

    private final PriceCalculator calculator = new PriceCalculator();

    @Test
    void 할인된_가격을_계산한다() {
        assertEquals(9_000, calculator.discount(10_000, 10));
    }
}
```

Spring Boot에서 다음 테스트는 애플리케이션 컨텍스트를 실행한다.

```
@SpringBootTest
class OrderServiceIntegrationTest {
}
```

`@SpringBootTest`는 실행이 상대적으로 느리고 많은 컴포넌트를 함께 사용하므로 단위 테스트와 성격이 다르다.

간단한 기준

- 한 클래스의 로직만 검증한다 → 가능한 한 순수 JUnit 단위 테스트
- Spring 설정, DB, 여러 컴포넌트의 연결을 함께 검증한다 → 통합 테스트 고려
- 모든 테스트에 무조건 `@SpringBootTest`를 붙이지 않는다

---

# JUnit과 Mockito의 역할 차이

JUnit은 테스트를 실행하고 결과를 검증한다.

```
@Test
void 주문한다() {
    // 실행과 assertion
}
```

Mockito는 테스트 대상이 의존하는 객체를 가짜로 만드는 데 사용한다.

```
OrderRepository repository = mock(OrderRepository.class);
```

역할을 구분하면 다음과 같다.

```
JUnit: 테스트 실행, 생명주기, assertion
Mockito: mock 객체와 행동 설정 및 호출 검증
AssertJ: 읽기 쉬운 assertion 문법
```

Mockito를 사용하지 않고 직접 만든 fake나 stub 객체를 사용할 수도 있다. “단위 테스트 = 반드시 Mockito”는 아니다.

---

# 자주 하는 실수

## 한 테스트에서 너무 많은 동작 검증하기

```
@Test
void 회원_전체_기능() {
    // 가입
    // 조회
    // 수정
    // 삭제
}
```

실패했을 때 어떤 동작이 잘못됐는지 찾기 어렵다. 의미 있는 동작 단위로 분리하는 편이 좋다.

## 테스트끼리 실행 순서에 의존하기

각 테스트는 단독 실행해도 성공해야 한다. `@Order`로 순서를 맞추는 것은 해결책이 아니다.

## 고정되지 않은 값 사용하기

```
assertEquals(LocalDateTime.now(), member.getCreatedAt());
```

두 `now()` 호출 사이에 시간이 지나 실패할 수 있다. 범위를 검증하거나 제어 가능한 시간을 주입하는 방식이 필요하다.

무작위 값, 현재 날짜, 타임존 등도 같은 문제를 만들 수 있다.

## `Thread.sleep()`으로 기다리기

```
Thread.sleep(1000);
```

테스트를 느리고 불안정하게 만든다. 비동기 처리가 완료됐는지 명확한 조건으로 확인하는 방법을 찾아야 한다.

## 구현 방법을 지나치게 검증하기

테스트는 “어떻게 구현했는가”보다 “어떤 결과를 내는가”를 검증해야 한다. 내부 구현에 너무 강하게 결합된 테스트는 리팩터링만 해도 깨진다.

## 테스트에서 실제 네트워크나 운영 DB 사용하기

외부 환경 상태에 따라 결과가 달라지고 실행도 느려진다. 이것은 보통 단위 테스트보다 통합 테스트 영역이다.

## Coverage 수치만 높이기

테스트 커버리지는 테스트되지 않은 코드를 발견하는 참고 지표이다. 커버리지가 높다고 테스트 품질이 반드시 높은 것은 아니다.

```
@Test
void 커버리지만_올리는_테스트() {
    service.execute(); // 아무것도 검증하지 않음
}
```