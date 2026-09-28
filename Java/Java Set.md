## Java `Set`

Java의 **`Set`은 중복된 원소를 허용하지 않는 컬렉션 인터페이스**이다. 수학의 집합과 비슷하게 생각하면 된다.

예를 들어 여러 사용자의 이메일을 저장할 때, 동일한 이메일을 한 번만 보관하고 싶다면 `Set`을 사용할 수 있다.

```
Set<String> emails = new HashSet<>();

emails.add("alice@example.com");
emails.add("bob@example.com");
emails.add("alice@example.com"); // 중복이므로 추가되지 않음

System.out.println(emails.size()); // 2
```

핵심은 **중복 제거**, **원소의 존재 여부 확인**, **집합 연산**이다.

---

## `Set`의 기본 특징

### 중복된 원소를 저장하지 않는다

이미 존재하는 원소를 다시 추가하면, 기존 원소를 유지하고 추가를 거절한다.

```
Set<String> names = new HashSet<>();

System.out.println(names.add("Java"));   // true
System.out.println(names.add("Python")); // true
System.out.println(names.add("Java"));   // false
```

`add()`의 반환값은 다음 의미를 가진다.

- `true`: 새로운 원소가 추가되어 집합이 변경됨
- `false`: 같은 원소가 이미 존재하여 집합이 변경되지 않음

따라서 중복을 먼저 검사할 필요 없이 다음처럼 작성할 수 있다.

```
if (!names.add("Java")) {
    System.out.println("이미 등록된 이름입니다.");
}
```

### 인덱스로 접근하지 않는다

`List`는 위치를 기준으로 원소에 접근할 수 있다.

```
List<String> list = List.of("A", "B", "C");

System.out.println(list.get(0)); // A
```

반면 `Set`에는 `get(int index)` 메서드가 없다.

```
Set<String> set = new HashSet<>();

set.add("A");
set.add("B");

// set.get(0); // 컴파일 오류
```

`Set`에서는 주로 “몇 번째 원소인가?”보다 **“이 원소가 존재하는가?”**를 다룬다.

```
System.out.println(set.contains("A")); // true
```

### 순서 보장 여부는 구현체에 따라 다르다

“`Set`은 순서가 없다”는 설명은 정확히는 부족하다.

|구현체|순서|
|---|---|
|`HashSet`|반복 순서를 보장하지 않음|
|`LinkedHashSet`|일반적인 `add()` 사용 시 삽입 순서 유지|
|`TreeSet`|정렬 순서 유지|

즉, **중복을 허용하지 않는 것은 공통이지만, 순서를 다루는 방식은 다르다.**

---

## `List`, `Set`, `Map`의 차이

|구분|`List`|`Set`|`Map`|
|---|---|---|---|
|저장 단위|원소|원소|키와 값|
|중복|원소 중복 허용|원소 중복 불가|키 중복 불가, 값 중복 허용|
|인덱스 접근|가능|불가능|인덱스 대신 키로 접근|
|순서|위치와 순서 유지|구현체에 따라 다름|구현체에 따라 다름|
|대표 구현체|`ArrayList`|`HashSet`|`HashMap`|

예를 들어 다음과 같이 선택할 수 있다.

```
// 방문 기록: 같은 페이지를 여러 번 방문할 수 있음
List<String> visitHistory = new ArrayList<>();

// 방문한 페이지 종류: 같은 페이지는 한 번만 저장
Set<String> visitedPages = new HashSet<>();

// 페이지별 방문 횟수: 페이지 → 횟수
Map<String, Integer> visitCounts = new HashMap<>();
```

구조적으로 `List`와 `Set`은 `Collection`을 상속하지만, `Map`은 별도의 인터페이스이다.

```
Iterable
└── Collection
    ├── List
    ├── Set
    └── Queue

Map
```

---

## 주요 메서드

|메서드|역할|반환값|
|---|---|---|
|`add(e)`|원소 추가|변경되면 `true`|
|`remove(o)`|같은 원소 제거|제거되면 `true`|
|`contains(o)`|원소 존재 여부 확인|존재하면 `true`|
|`size()`|원소 개수 확인|`int`|
|`isEmpty()`|비어 있는지 확인|비어 있으면 `true`|
|`clear()`|모든 원소 제거|없음|
|`addAll(c)`|다른 컬렉션의 원소 추가|변경되면 `true`|
|`containsAll(c)`|다른 컬렉션의 원소를 모두 포함하는지 확인|모두 포함하면 `true`|
|`retainAll(c)`|다른 컬렉션에도 있는 원소만 유지|변경되면 `true`|
|`removeAll(c)`|다른 컬렉션에 있는 원소 제거|변경되면 `true`|

기본적인 사용 예시이다.

```
import java.util.HashSet;
import java.util.Set;

public class Main {
    public static void main(String[] args) {
        Set<String> languages = new HashSet<>();

        languages.add("Java");
        languages.add("Python");
        languages.add("JavaScript");

        System.out.println(languages.contains("Java")); // true
        System.out.println(languages.size());           // 3

        languages.remove("Python");

        System.out.println(languages.size());           // 2

        for (String language : languages) {
            System.out.println(language);
        }

        languages.clear();

        System.out.println(languages.isEmpty());        // true
    }
}
```

`HashSet`이므로 반복문의 출력 순서는 보장되지 않는다.

---

## 주요 구현체 비교

### `HashSet`: 일반적인 기본 선택

`HashSet`은 해시 테이블을 기반으로 원소를 저장한다. 내부적으로는 `HashMap`을 사용하며, 집합의 원소를 맵의 키로 관리한다.

```
Set<Integer> numbers = new HashSet<>();

numbers.add(30);
numbers.add(10);
numbers.add(20);
numbers.add(10);

System.out.println(numbers.size()); // 3
```

특징은 다음과 같다.

- 중복을 허용하지 않는다.
- 반복 순서를 보장하지 않는다.
- `null`을 하나 저장할 수 있다.
- 해시가 적절히 분산된다는 전제에서 추가·삭제·검색이 평균적으로 빠르다.

**순서가 필요 없고 중복 제거나 존재 여부 확인이 목적이라면 우선 고려할 구현체이다.**

> 출력이 우연히 정렬된 것처럼 보여도 그 순서에 의존하면 안 된다.

### `LinkedHashSet`: 중복 제거와 삽입 순서 유지

`LinkedHashSet`은 해시 구조에 원소의 순서를 관리하는 연결 구조를 더한 구현체이다.

```
Set<String> names = new LinkedHashSet<>();

names.add("민수");
names.add("지연");
names.add("서준");
names.add("민수");

System.out.println(names);
// [민수, 지연, 서준]
```

일반적인 `add()`로 기존 원소를 다시 추가해도 그 원소의 위치는 바뀌지 않는다.

특징은 다음과 같다.

- 중복을 허용하지 않는다.
- 삽입 순서를 유지한다.
- `null`을 하나 저장할 수 있다.
- 순서 관리 때문에 `HashSet`보다 추가 메모리가 필요한다.

**원본 데이터에서 처음 등장한 순서를 유지하면서 중복을 제거할 때 유용하다.**

```
List<String> original = List.of("B", "A", "B", "C", "A");

Set<String> unique = new LinkedHashSet<>(original);

System.out.println(unique); // [B, A, C]
```

### `TreeSet`: 정렬된 집합

`TreeSet`은 트리 구조를 사용하며, 원소를 정렬된 상태로 유지한다.

```
Set<Integer> numbers = new TreeSet<>();

numbers.add(30);
numbers.add(10);
numbers.add(20);
numbers.add(10);

System.out.println(numbers);
// [10, 20, 30]
```

기본적으로 원소의 자연 순서를 사용한다.

- 정수: 숫자 크기순
- 문자열: `String`이 정의한 사전식 순서
- 사용자 정의 객체: `Comparable` 구현 또는 별도의 `Comparator` 필요

내림차순으로 정렬할 수도 있다.

```
Set<Integer> descending =
        new TreeSet<>(Comparator.reverseOrder());

descending.addAll(List.of(30, 10, 20));

System.out.println(descending);
// [30, 20, 10]
```

`TreeSet`의 정렬 관련 기능을 사용하려면 `NavigableSet` 타입으로 선언하는 것이 편리한다.

```
NavigableSet<Integer> scores =
        new TreeSet<>(List.of(60, 70, 80, 90));

System.out.println(scores.first());     // 60
System.out.println(scores.last());      // 90
System.out.println(scores.lower(80));   // 70: 80보다 작은 값 중 최댓값
System.out.println(scores.floor(80));   // 80: 80 이하 중 최댓값
System.out.println(scores.ceiling(75)); // 80: 75 이상 중 최솟값
System.out.println(scores.higher(80));  // 90: 80보다 큰 값 중 최솟값
```

자연 순서를 사용하는 `TreeSet`은 `null`을 허용하지 않는다. 별도 비교자가 `null` 비교를 지원하면 허용할 수 있지만, 일반적으로는 `null`을 넣지 않는 방식으로 사용하는 것이 간단하다.

### 구현체 비교표

|항목|`HashSet`|`LinkedHashSet`|`TreeSet`|
|---|---|---|---|
|핵심 구조|해시 테이블|해시 테이블 + 연결 구조|균형 이진 탐색 트리|
|순서|보장 없음|삽입 순서|정렬 순서|
|중복 판단|`hashCode()`와 `equals()`|`hashCode()`와 `equals()`|비교 결과가 `0`인지 확인|
|추가·삭제·검색|평균 `O(1)`|평균 `O(1)`|`O(log n)`|
|`null`|하나 허용|하나 허용|자연 순서에서는 불가|
|주요 용도|빠른 중복 검사|순서를 유지하는 중복 제거|정렬과 범위 탐색|

해시 기반 구현체의 `O(1)`은 평균적인 성능이며, 해시 분포 등에 따라 달라질 수 있다.

---

## 가장 중요한 원리: 무엇을 “중복”이라고 판단할까?

### `HashSet`은 `equals()`와 `hashCode()`를 함께 사용한다

개념적으로 다음 과정을 거친다.

1. `hashCode()`로 원소를 찾을 위치를 좁힌다.
2. 해당 위치의 후보들과 `equals()`로 동등한지 확인한다.
3. 동등한 원소가 있으면 새 원소를 추가하지 않는다.

여기서 반드시 지켜야 할 규칙이 있다.

> **`equals()`가 `true`인 두 객체는 반드시 같은 `hashCode()`를 반환해야 한다.**

반대는 성립하지 않는다.

> 같은 `hashCode()`를 가진 객체라도 `equals()`는 `false`일 수 있다.

이 경우를 **해시 충돌**이라고 하며, 해시값이 같다는 이유만으로 중복 처리하지는 않는다.

### 사용자 정의 클래스에서는 중복 기준을 직접 정의해야 한다

다음 클래스는 `equals()`와 `hashCode()`를 재정의하지 않는다.

```
class User {
    private final String email;

    User(String email) {
        this.email = email;
    }
}
```

이 상태에서 두 객체를 추가하면 다음과 같다.

```
Set<User> users = new HashSet<>();

users.add(new User("alice@example.com"));
users.add(new User("alice@example.com"));

System.out.println(users.size()); // 2
```

내용이 같아 보여도 서로 다른 객체이기 때문이다. `Object`에서 상속한 기본 `equals()`는 같은 객체인지를 비교한다.

이메일이 같으면 같은 사용자로 취급하려면 다음처럼 작성할 수 있다.

```
import java.util.Objects;

final class User {
    private final String email;

    User(String email) {
        this.email = email;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) {
            return true;
        }

        if (!(obj instanceof User)) {
            return false;
        }

        User other = (User) obj;
        return Objects.equals(email, other.email);
    }

    @Override
    public int hashCode() {
        return Objects.hash(email);
    }
}
```

이제 같은 이메일의 객체는 하나만 저장된다.

```
Set<User> users = new HashSet<>();

users.add(new User("alice@example.com"));
users.add(new User("alice@example.com"));

System.out.println(users.size()); // 1
```

**`equals()`를 재정의했다면 `hashCode()`도 함께 재정의하는 것이 원칙이다.**

### `TreeSet`은 비교 결과가 `0`이면 중복으로 처리한다

이 차이는 실수하기 쉽다.

```
Set<String> words =
        new TreeSet<>(Comparator.comparingInt(String::length));

words.add("cat");
words.add("dog");
words.add("apple");

System.out.println(words);
// [cat, apple]
```

`"cat"`과 `"dog"`는 서로 다른 문자열이지만 길이가 같다. 비교자가 두 값을 비교하면 `0`을 반환하므로, `TreeSet`은 같은 원소로 취급한다.

길이가 같은 서로 다른 문자열도 보관하려면 추가 비교 기준을 지정해야 한다.

```
Set<String> words = new TreeSet<>(
        Comparator.comparingInt(String::length)
                  .thenComparing(Comparator.naturalOrder())
);

words.addAll(List.of("cat", "dog", "apple"));

System.out.println(words);
// [cat, dog, apple]
```

일반적인 `Set`의 동등성 규약을 지키려면 **비교 결과가 `0`인 경우와 `equals()`가 `true`인 경우가 일치하도록** 설계해야 한다.

---

## 합집합·교집합·차집합 구하기

다음 두 집합이 있다고 가정한다.

```
Set<Integer> a = new HashSet<>(List.of(1, 2, 3));
Set<Integer> b = new HashSet<>(List.of(3, 4, 5));
```

### 합집합: 어느 한쪽에라도 있는 원소

`addAll()`을 사용한다.

```
Set<Integer> union = new HashSet<>(a);
union.addAll(b);

// 원소: 1, 2, 3, 4, 5
```

### 교집합: 양쪽에 모두 있는 원소

`retainAll()`을 사용한다.

```
Set<Integer> intersection = new HashSet<>(a);
intersection.retainAll(b);

// 원소: 3
```

### 차집합: A에는 있고 B에는 없는 원소

`removeAll()`을 사용한다.

```
Set<Integer> difference = new HashSet<>(a);
difference.removeAll(b);

// 원소: 1, 2
```

### 부분집합 여부 확인

A가 B의 부분집합인지는 B가 A의 모든 원소를 포함하는지 확인하면 된다.

```
boolean aIsSubsetOfB = b.containsAll(a);

System.out.println(aIsSubsetOfB); // false
```

**`addAll()`, `retainAll()`, `removeAll()`은 호출 대상 집합을 직접 변경한다.** 원본을 보존하려면 위 예제처럼 복사본을 만들어야 한다.

---

## 수정할 수 없는 `Set`

### `Set.of()`: 고정된 값으로 만들기

Java 9부터 사용할 수 있다.

```
Set<String> roles = Set.of("ADMIN", "USER", "GUEST");
```

생성 후 원소를 추가하거나 제거할 수 없다.

```
roles.add("MANAGER"); // UnsupportedOperationException
```

또한 다음 제약이 있다.

```
Set.of("A", "A"); // IllegalArgumentException: 중복 원소
Set.of("A", null); // NullPointerException
```

주의할 점은 **`HashSet.add()`와 달리, 중복을 조용히 무시하지 않고 예외를 발생시킨다**는 것이다. 반복 순서도 보장하지 않는다.

### `Set.copyOf()`: 기존 컬렉션으로 만들기

Java 10부터 사용할 수 있다.

```
List<String> names = List.of("A", "B", "A");

Set<String> uniqueNames = Set.copyOf(names);

System.out.println(uniqueNames.size()); // 2
```

- 수정할 수 없는 집합을 반환한다.
- 입력 컬렉션의 중복 원소는 하나로 합쳐진다.
- `null` 원소는 허용하지 않는다.
- 원본 컬렉션의 이후 추가·삭제는 결과에 반영되지 않는다.

### `Collections.unmodifiableSet()`: 수정 불가 뷰 만들기

```
Set<String> original = new HashSet<>();
original.add("A");

Set<String> view = Collections.unmodifiableSet(original);

original.add("B");

System.out.println(view.contains("B")); // true
```

`view`를 통해 수정할 수는 없지만, **원본 집합의 변경은 보인다.**

| 방식                                      | 원본의 이후 추가·삭제 반영 |
| --------------------------------------- | --------------- |
| `Set.copyOf(original)`                  | 반영되지 않음         |
| `Collections.unmodifiableSet(original)` | 반영됨             |

다만 두 방식 모두 **원소 객체 자체를 깊게 복사하거나 불변으로 만들지는 않는다.** 원소가 변경 가능한 객체라면 그 객체의 내부 상태는 여전히 바뀔 수 있다.

---

## 자주 발생하는 실수

### 저장한 뒤 중복 판단에 사용하는 필드를 변경하기

예를 들어 `email`을 기준으로 `equals()`와 `hashCode()`를 구현한 객체를 `HashSet`에 넣었다고 가정한다.

```
users.add(user);

// email 변경으로 hashCode()도 달라지는 상황
user.setEmail("changed@example.com");
```

객체는 이전 해시값을 기준으로 저장되어 있는데, 검색은 변경된 해시값을 기준으로 수행될 수 있다.

그 결과 다음과 같은 문제가 생길 수 있다.

- 실제 저장된 객체를 `contains()`로 찾지 못함
- `remove()`로 제거하지 못함
- 의도한 중복 방지 규칙이 깨짐

`TreeSet`도 정렬 기준 필드를 변경하면 트리 안의 배치와 실제 값이 맞지 않게 된다.

**집합에 저장하는 동안 동등성·해시·정렬에 사용하는 필드는 바꾸지 않는 것이 좋다.** 꼭 바꿔야 한다면 변경 전에 제거하고, 변경 후 다시 추가해야 한다.

### 순회 중 `set.remove()` 호출하기

일반적인 `HashSet`에서 다음 코드는 `ConcurrentModificationException`을 일으킬 수 있다.

```
for (String value : set) {
    if (value.startsWith("A")) {
        set.remove(value); // 잘못된 순회 중 변경 방식
    }
}
```

조건에 맞는 원소를 지우려면 `removeIf()`가 간단한다.

```
set.removeIf(value -> value.startsWith("A"));
```

직접 반복자를 사용할 경우에는 `Iterator.remove()`를 사용한다.

```
Iterator<String> iterator = set.iterator();

while (iterator.hasNext()) {
    String value = iterator.next();

    if (value.startsWith("A")) {
        iterator.remove();
    }
}
```

### `Set`을 무조건 스레드 안전하다고 생각하기

`HashSet`, `LinkedHashSet`, `TreeSet`은 기본적으로 스레드 안전하지 않다.

여러 스레드가 동시에 접근하고 변경해야 한다면 동시성용 구현을 고려할 수 있다.

```
Set<String> concurrentSet = ConcurrentHashMap.newKeySet();
```

이 집합은 `null`을 허용하지 않다.

단, 동시성 컬렉션을 사용해도 여러 메서드 호출을 조합한 작업 전체가 자동으로 원자적이 되는 것은 아니다. 중복 확인 후 추가가 목적이라면 `contains()`와 `add()`를 나누기보다 `add()`의 반환값을 활용하는 것이 적절하다.

---

## 활용 예시

### 순서를 유지하면서 `List`의 중복 제거

```
List<String> original = List.of("Java", "Python", "Java", "Go");

List<String> unique =
        new ArrayList<>(new LinkedHashSet<>(original));

System.out.println(unique);
// [Java, Python, Go]
```

### 처리한 식별자 기록

```
Set<Long> processedIds = new HashSet<>();

for (Long id : List.of(101L, 102L, 101L, 103L)) {
    if (processedIds.add(id)) {
        System.out.println("처리: " + id);
    }
}
```

출력:

```
처리: 101
처리: 102
처리: 103
```

### 열거형 값 관리: `EnumSet`

원소가 모두 같은 `enum` 타입이라면 전용 구현체인 `EnumSet`도 좋은 선택이다.

```
enum Permission {
    READ, WRITE, DELETE
}

Set<Permission> permissions =
        EnumSet.of(Permission.READ, Permission.WRITE);

System.out.println(permissions.contains(Permission.READ));   // true
System.out.println(permissions.contains(Permission.DELETE)); // false
```

`EnumSet`은 열거형을 비트 벡터 형태로 관리하여 효율적이며, 반복 순서는 열거형 상수의 선언 순서이다.

---

## 어떤 `Set`을 선택하면 될까?

| 필요한 동작                  | 선택                              |
| ----------------------- | ------------------------------- |
| 순서 없이 중복 제거와 빠른 조회      | `HashSet`                       |
| 처음 등장한 순서를 유지하며 중복 제거   | `LinkedHashSet`                 |
| 정렬 유지, 최솟값·최댓값·인접 값 탐색  | `TreeSet`                       |
| 열거형 상수의 집합              | `EnumSet`                       |
| 소수의 고정된 값으로 수정 불가 집합 생성 | `Set.of()`                      |
| 기존 컬렉션에서 수정 불가 집합 생성    | `Set.copyOf()`                  |
| 여러 스레드가 동시에 변경하는 집합     | `ConcurrentHashMap.newKeySet()` |

`Set`을 제대로 사용하려면 두 가지를 먼저 정하면 된다. **어떤 원소를 같다고 볼 것인지**, 그리고 **어떤 순서가 필요한지**이다. 이 두 기준이 명확하면 구현체 선택과 중복 처리 방식도 자연스럽게 결정된다.