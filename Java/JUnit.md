# JUnit

JUnit은 Java/JVM 애플리케이션의 동작을 자동으로 검증하는 테스트 프레임워크이다. 메서드를 실행한 뒤 실제 결과와 기대 결과가 같은지 확인하고, 테스트 결과를 성공·실패·중단·비활성 상태로 보고한다.

현재 안정 버전은 **JUnit 6.1.2**이며 Java 17 이상에서 실행된다. 기존 자료에서 흔히 보는 “JUnit 5”와 마찬가지로 Platform·Jupiter·Vintage 구조를 사용한다. Java 8이나 11 프로젝트라면 JUnit 5.x를 사용해야 한다.

---

## JUnit의 목적

예를 들어 다음 코드가 있다고 하면

```
class Calculator {

    int add(int a, int b) {
        return a + b;
    }

    int divide(int a, int b) {
        if (b == 0) {
            throw new IllegalArgumentException("0으로 나눌 수 없습니다.");
        }
        return a / b;
    }
}
```

사람이 매번 직접 실행해 결과를 확인하는 대신 테스트 코드로 동작을 명세한다.

```
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;

import org.junit.jupiter.api.Test;

class CalculatorTest {

    private final Calculator calculator = new Calculator();

    @Test
    void 두_수를_더한다() {
        int result = calculator.add(2, 3);

        assertEquals(5, result);
    }

    @Test
    void 영으로_나누면_예외가_발생한다() {
        IllegalArgumentException exception =
                assertThrows(
                        IllegalArgumentException.class,
                        () -> calculator.divide(10, 0)
                );

        assertEquals("0으로 나눌 수 없습니다.", exception.getMessage());
    }
}
```

이렇게 해두면 구현을 변경한 뒤에도 기존 기능이 깨지지 않았는지 빠르게 확인할 수 있다.

JUnit 테스트는 보통 다음 세 단계로 작성한다.

```
Given / Arrange  : 테스트 조건과 객체 준비
When / Act       : 검증할 동작 실행
Then / Assert    : 결과 확인
```

---

## JUnit의 구조

JUnit 5 이후에는 하나의 라이브러리가 아니라 세 부분으로 구성된다.

```
Gradle / Maven / IDE
        ↓
JUnit Platform
        ↓
TestEngine
  ├─ Jupiter Engine → JUnit 5·6 테스트
  ├─ Vintage Engine → JUnit 3·4 테스트
  └─ 기타 엔진       → 다른 테스트 프레임워크
```

### JUnit Platform

테스트를 발견하고 실행하는 기반이다.

- IDE, Gradle, Maven과 테스트 엔진을 연결
- 테스트 클래스와 메서드 검색
- 실행 결과와 리포트 제공
- 다른 테스트 프레임워크가 사용할 수 있는 `TestEngine` API 제공

### JUnit Jupiter

우리가 일반적으로 “JUnit 테스트를 작성한다”고 할 때 사용하는 API이다.

- `@Test`
- `@BeforeEach`
- `Assertions`
- 파라미터 테스트
- 확장 모델인 `Extension`

패키지 이름은 대부분 `org.junit.jupiter.*`이다.

### JUnit Vintage

JUnit 3·4 테스트를 JUnit Platform에서 실행하기 위한 호환 엔진이다. 현재는 deprecated 상태이므로, 기존 테스트를 옮기는 동안 임시로 사용하는 것이 권장된다.

---

## 프로젝트 설정

일반적인 디렉터리 구조는 다음과 같다.

```
src/
├─ main/
│  └─ java/
│     └─ Calculator.java
└─ test/
   └─ java/
      └─ CalculatorTest.java
```

### Gradle

```
plugins {
    id 'java'
}

repositories {
    mavenCentral()
}

dependencies {
    testImplementation(platform("org.junit:junit-bom:6.1.2"))
    testImplementation("org.junit.jupiter:junit-jupiter")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

test {
    useJUnitPlatform()
}
```

실행:

```
./gradlew test
```

JUnit 6은 Maven Surefire/Failsafe 3.0.0 이상이 필요하다.

Spring Boot 프로젝트라면 대개 `spring-boot-starter-test`가 JUnit Jupiter를 포함하므로 별도로 버전을 추가하기 전에 Boot의 의존성 관리를 확인해야 한다.

---

## 주요 애너테이션

### 기본 애너테이션

|애너테이션|의미|
|---|---|
|`@Test`|일반 테스트 메서드|
|`@DisplayName`|사람이 읽기 쉬운 테스트 이름|
|`@Disabled`|테스트를 임시로 실행하지 않음|
|`@BeforeEach`|각 테스트 실행 전 호출|
|`@AfterEach`|각 테스트 실행 후 호출|
|`@BeforeAll`|클래스의 모든 테스트 실행 전 한 번 호출|
|`@AfterAll`|클래스의 모든 테스트 실행 후 한 번 호출|
|`@Nested`|테스트를 계층적으로 그룹화|
|`@Tag`|테스트 분류 및 선택 실행|
|`@Timeout`|테스트 실행 제한 시간 지정|

### 실행 순서

```
@BeforeAll

  @BeforeEach
  @Test
  @AfterEach

  @BeforeEach
  @Test
  @AfterEach

@AfterAll
```

예제:

```
import org.junit.jupiter.api.*;

class UserServiceTest {

    @BeforeAll
    static void beforeAll() {
        System.out.println("전체 테스트 시작");
    }

    @BeforeEach
    void setUp() {
        System.out.println("테스트 데이터 준비");
    }

    @Test
    void testA() {
    }

    @Test
    void testB() {
    }

    @AfterEach
    void tearDown() {
        System.out.println("테스트 데이터 정리");
    }

    @AfterAll
    static void afterAll() {
        System.out.println("전체 테스트 종료");
    }
}
```

기본 테스트 인스턴스 생명주기는 `PER_METHOD`이다. 즉, 테스트 메서드마다 새로운 `UserServiceTest` 객체가 만들어져 테스트 간 상태 오염을 줄인다.

```
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class UserServiceTest {
}
```

`PER_CLASS`를 사용하면 모든 테스트가 같은 인스턴스를 공유하고 `@BeforeAll`과 `@AfterAll`을 비정적 메서드로 작성할 수 있다. 대신 필드 상태가 테스트 사이에 남을 수 있으므로 주의해야 한다.

---

## Assertion

Assertion은 “이 결과가 기대한 조건을 만족하는가?”를 검증한다.

```
import static org.junit.jupiter.api.Assertions.*;
```

### 값 검증

```
assertEquals(10, actual);       // 값이 같음
assertNotEquals(0, actual);     // 값이 다름

assertTrue(result > 0);
assertFalse(list.isEmpty());

assertNull(value);
assertNotNull(value);
```

`assertEquals()`의 순서는 기대값, 실제값이다.

```
assertEquals(expected, actual);
```

### 객체와 컬렉션

```
assertSame(expectedObject, actualObject);
assertNotSame(expectedObject, actualObject);

assertArrayEquals(
        new int[]{1, 2, 3},
        actualArray
);

assertIterableEquals(
        List.of("A", "B"),
        actualList
);
```

- `assertEquals()`는 보통 객체의 `equals()`를 사용한다.
- `assertSame()`은 두 변수가 완전히 같은 인스턴스를 가리키는지 확인한다.

### 여러 조건 함께 검증

```
assertAll(
        () -> assertEquals("홍길동", user.getName()),
        () -> assertEquals(20, user.getAge()),
        () -> assertTrue(user.isActive())
);
```

일반 assertion을 순서대로 작성하면 첫 번째 실패에서 테스트가 종료된다. `assertAll()`은 가능한 검증을 모두 실행하고 실패들을 함께 보여준다.

### 예외 검증

```
IllegalArgumentException exception =
        assertThrows(
                IllegalArgumentException.class,
                () -> calculator.divide(10, 0)
        );

assertEquals(
        "0으로 나눌 수 없습니다.",
        exception.getMessage()
);
```

차이점은 다음과 같다.

```
assertThrows(RuntimeException.class, executable);
```

하위 타입 예외도 허용한다.

```
assertThrowsExactly(RuntimeException.class, executable);
```

정확히 `RuntimeException` 타입이어야 한다.

예외가 발생하지 않아야 한다면 다음을 사용한다.

```
assertDoesNotThrow(() -> service.process());
```

### 시간 검증

```
import static java.time.Duration.ofSeconds;

assertTimeout(
        ofSeconds(1),
        () -> service.process()
);
```

`assertTimeout()`은 같은 스레드에서 작업이 끝날 때까지 기다린 뒤 제한 시간을 초과했는지 판단한다.

```
assertTimeoutPreemptively(
        ofSeconds(1),
        () -> service.process()
);
```

`assertTimeoutPreemptively()`는 다른 스레드에서 실행하고 시간 초과 시 중단을 시도한다. 따라서 `ThreadLocal`이나 Spring 테스트 트랜잭션 컨텍스트가 전달되지 않는 문제가 생길 수 있어 신중하게 사용해야 한다.

### 실패 메시지

```
assertEquals(
        expected,
        actual,
        () -> "사용자 ID " + userId + "의 상태가 올바르지 않습니다."
);
```

메시지 공급자 `() -> ...`를 사용하면 테스트가 실패할 때만 메시지를 계산한다.

---

## Assumption

Assertion이 틀리면 테스트가 실패한다. Assumption이 충족되지 않으면 테스트는 실패가 아니라 중단, 즉 aborted 처리된다.

```
import static org.junit.jupiter.api.Assumptions.assumeTrue;

@Test
void 리눅스에서만_실행한다() {
    assumeTrue(
            System.getProperty("os.name").toLowerCase().contains("linux")
    );

    // 리눅스에서만 의미 있는 검증
}
```

환경이 맞을 때만 실행해야 하는 테스트에 적합한다. 그러나 핵심 비즈니스 조건을 assumption으로 건너뛰면 결함을 놓칠 수 있다.

조건부 애너테이션도 사용할 수 있다.

```
@EnabledOnOs(OS.LINUX)
@Test
void linuxOnly() {
}
```

---

## 파라미터 테스트

같은 동작을 여러 입력으로 검증할 때 사용한다.

### `@ValueSource`

```
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

@ParameterizedTest
@ValueSource(strings = {"level", "radar", "racecar"})
void 회문을_판별한다(String value) {
    assertTrue(Palindrome.isPalindrome(value));
}
```

이 테스트는 입력값마다 한 번씩, 총 세 번 실행된다.

### `@CsvSource`

```
@ParameterizedTest(name = "{0} + {1} = {2}")
@CsvSource({
        "1, 2, 3",
        "5, 7, 12",
        "-1, 1, 0"
})
void 덧셈을_검증한다(int a, int b, int expected) {
    assertEquals(expected, calculator.add(a, b));
}
```

### `@MethodSource`

복잡한 객체를 전달할 때 유용하다.

```
import java.util.stream.Stream;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.MethodSource;

@ParameterizedTest
@MethodSource("userData")
void 성인_여부를_판단한다(User user, boolean expected) {
    assertEquals(expected, user.isAdult());
}

static Stream<Arguments> userData() {
    return Stream.of(
            Arguments.of(new User("철수", 20), true),
            Arguments.of(new User("영희", 17), false)
    );
}
```

주요 데이터 소스는 다음과 같다.

- `@ValueSource`: 단순한 값 한 개
- `@NullSource`: `null`
- `@EmptySource`: 빈 문자열이나 빈 컬렉션
- `@NullAndEmptySource`: `null`과 빈 값
- `@EnumSource`: enum 값
- `@CsvSource`: 코드에 직접 작성한 CSV
- `@CsvFileSource`: 파일로 관리하는 CSV
- `@MethodSource`: 메서드가 생성하는 데이터

JUnit 6은 `@ParameterizedClass`를 사용해 클래스 전체를 여러 입력으로 반복 실행하는 것도 지원다.

---

## 반복·중첩·동적 테스트

### 반복 테스트

```
@RepeatedTest(5)
void 랜덤_값은_항상_범위_안에_있다() {
    int value = randomService.nextInt(1, 10);

    assertTrue(value >= 1 && value <= 10);
}
```

단순히 반복한다고 테스트가 신뢰성 있어지는 것은 아니다. 랜덤 테스트가 실패하면 입력값이나 seed를 기록해 재현 가능하게 만드는 것이 중요하다.

### 중첩 테스트

```
@DisplayName("계좌")
class AccountTest {

    @Nested
    @DisplayName("출금 시")
    class Withdraw {

        @Test
        void 잔액이_충분하면_출금된다() {
        }

        @Test
        void 잔액이_부족하면_실패한다() {
        }
    }
}
```

관련 시나리오를 계층적으로 표현할 수 있어 테스트 리포트가 읽기 쉬워진다.

### 동적 테스트

```
@TestFactory
Stream<DynamicTest> dynamicTests() {
    return Stream.of(1, 2, 3)
            .map(value -> DynamicTest.dynamicTest(
                    value + "은 양수다",
                    () -> assertTrue(value > 0)
            ));
}
```

`@TestFactory`는 실행 시점에 테스트를 생성한다. 일반적인 입력 조합은 `@ParameterizedTest`가 더 간단하고, 테스트 수나 구조 자체가 런타임 데이터로 결정될 때 동적 테스트가 적합하다.

---

## 태그와 선택 실행

```
@Tag("unit")
@Test
void 빠른_단위_테스트() {
}

@Tag("integration")
@Test
void 데이터베이스_연동_테스트() {
}
```

Gradle:

```
test {
    useJUnitPlatform {
        includeTags "unit"
        excludeTags "slow"
    }
}
```

태그를 이용하면 CI에서 빠른 테스트와 느린 테스트를 분리할 수 있다.

```
커밋/PR       → unit 테스트
배포 전       → unit + integration 테스트
야간 빌드     → unit + integration + slow 테스트
```

---

## Extension 모델

JUnit Jupiter에서는 JUnit 4의 `Runner`와 `Rule`을 Extension으로 통합했다.

```
@ExtendWith(MyExtension.class)
class ServiceTest {
}
```

Extension으로 다음 작업을 구현할 수 있다.

- 테스트 실행 전후 공통 처리
- 테스트 인스턴스 생성 및 후처리
- 메서드 파라미터 주입
- 조건에 따른 테스트 실행
- 실패 결과 기록
- 외부 자원 생성과 정리

대표적인 확장 예가 Mockito이다.

```
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    PaymentClient paymentClient;

    @InjectMocks
    OrderService orderService;
}
```

JUnit 자체는 테스트 실행과 assertion을 담당한다. Mock 객체 생성은 Mockito 같은 별도 라이브러리가 담당한다.

---

## JUnit 4와 Jupiter 비교

|JUnit 4|JUnit Jupiter|
|---|---|
|`org.junit.Test`|`org.junit.jupiter.api.Test`|
|`@Before`|`@BeforeEach`|
|`@After`|`@AfterEach`|
|`@BeforeClass`|`@BeforeAll`|
|`@AfterClass`|`@AfterAll`|
|`@Ignore`|`@Disabled`|
|`@RunWith`|`@ExtendWith`|
|`@Rule`, `@ClassRule`|Extension|
|Test(expected=...)|`assertThrows()`|
|Test(timeout=...)|`@Timeout`, `assertTimeout()`|

주의할 점은 import이다.

```
// JUnit 4
import org.junit.Test;

// Jupiter
import org.junit.jupiter.api.Test;
```

잘못된 `@Test`를 import하면 테스트가 발견되지 않거나 Jupiter 기능을 사용할 수 없다.

기존 JUnit 4 테스트는 Vintage Engine으로 잠시 함께 실행할 수 있지만, 신규 테스트는 Jupiter로 작성하고 점진적으로 이전하는 편이 좋다.

---

## 단위 테스트와 통합 테스트

JUnit은 단위 테스트만을 위한 도구는 아니다.

### 단위 테스트

- 한 클래스나 작은 단위 검증
- 실제 DB, 네트워크, 파일 시스템을 가급적 사용하지 않음
- 빠르고 독립적
- 필요하면 의존성을 mock으로 대체

```
class OrderServiceTest {
    // PaymentClient를 mock으로 대체
}
```

### 통합 테스트

- 여러 컴포넌트의 연동 검증
- 실제 또는 테스트용 DB 사용
- Spring Context, HTTP, 메시지 브로커 등을 포함할 수 있음
- 단위 테스트보다 느림

```
@SpringBootTest
class OrderIntegrationTest {
}
```

둘 다 JUnit 위에서 실행할 수 있지만 테스트 목적과 범위가 다르다.

---

## 좋은 테스트

좋은 테스트는 다음 성질을 가진다.

- 독립적이다: 다른 테스트 실행 여부나 순서에 의존하지 않음
- 반복 가능하다: 같은 조건에서는 항상 같은 결과
- 빠르다: 개발 중 자주 실행할 수 있음
- 의미가 분명하다: 무엇이 실패했는지 이름만 보고 파악 가능
- 구현이 아니라 동작을 검증한다
- 외부 시간, 네트워크, 랜덤 값에 무분별하게 의존하지 않는다

좋은 이름의 예:

```
@Test
void 잔액보다_큰_금액을_출금하면_예외가_발생한다() {
}
```

또는 영어 관례:

```
@Test
void withdraw_throwsException_whenAmountExceedsBalance() {
}
```

---

## 안 좋은 테스트

1. 테스트 순서에 의존하기

```
@Test
void 먼저_사용자를_등록한다() {}

@Test
void 그_사용자를_조회한다() {}
```

각 테스트는 독립적으로 데이터를 준비해야 한다.

2. 실제 시간에 의존하기

```
LocalDateTime.now()
```

`Clock`을 주입하면 고정된 시간으로 테스트할 수 있다.

3. `Thread.sleep()` 사용하기

테스트가 느리고 불안정해진다. 비동기 테스트는 조건 대기 도구나 명확한 동기화 방식을 사용하는 편이 좋다.

4. 지나치게 많은 것을 한 테스트에서 검증하기

한 테스트가 회원가입, 로그인, 결제, 환불까지 모두 수행하면 실패 원인을 찾기 어렵다.

5. assertion이 없는 테스트

예외가 발생하지 않았다는 사실만으로 비즈니스 결과가 올바르다고 할 수 없다.

6. 코드 커버리지만 목표로 삼기

커버리지 100%여도 경계값이나 오류 조건을 제대로 검증하지 않았다면 좋은 테스트가 아니다.