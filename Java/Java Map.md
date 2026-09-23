# Java의 Map

`Map`은 데이터를 **키(key)와 값(value)의 쌍**으로 저장하는 자료구조이다.

예를 들어 이름으로 전화번호를 찾거나, 상품 코드로 상품 정보를 조회할 때 사용한다.

```
Map<String, String> phoneBook = new HashMap<>();

phoneBook.put("김철수", "010-1234-5678");
phoneBook.put("이영희", "010-9876-5432");

System.out.println(phoneBook.get("김철수"));
// 010-1234-5678
```

여기서 `"김철수"`는 **키**, `"010-1234-5678"`은 **값**이다.

---

## Map의 핵심 특징

### 키는 중복될 수 없고, 값은 중복될 수 있다

하나의 Map에서 같은 키는 하나만 존재한다. 기존 키로 값을 다시 저장하면 이전 값을 교체한다.

```
Map<String, Integer> scores = new HashMap<>();

scores.put("철수", 80);
scores.put("영희", 80); // 값의 중복은 허용
scores.put("철수", 95); // 기존 값 80을 95로 교체

System.out.println(scores.get("철수")); // 95
System.out.println(scores.size());      // 2
```

**키의 유일성은 Map 내부에서만 적용된다.** 다른 Map에는 같은 키가 있어도 문제없다.

### 순서와 null 허용 여부는 구현체에 따라 다르다

`Map` 인터페이스 자체가 저장 순서나 `null` 허용 여부를 일괄적으로 정하지는 않는다.

- `HashMap`: 순서를 보장하지 않으며, `null` 키와 값을 허용한다.
- `LinkedHashMap`: 기본적으로 삽입 순서를 유지한다.
- `TreeMap`: 키를 정렬해서 관리한다.
- `ConcurrentHashMap`: 여러 스레드의 동시 접근을 지원하며, `null` 키와 값을 허용하지 않는다.

### Map은 Collection을 상속하지 않는다

Java 컬렉션 프레임워크에 속하지만, `Map`은 `Collection`의 하위 인터페이스가 아니다.

```
Collection
├── List
├── Set
└── Queue

Map
├── HashMap
│   └── LinkedHashMap
├── TreeMap
└── ConcurrentHashMap
```

위 그림은 주요 구현체의 관계를 단순화한 것이다.

`List`와 `Set`은 개별 요소를 저장하지만, `Map`은 키와 값의 대응 관계를 저장한다.

---

## Map 선언과 생성

일반적으로 변수 타입은 인터페이스인 `Map`으로 선언하고, 실제 객체는 구현체로 생성한다.

```
import java.util.HashMap;
import java.util.Map;

Map<String, Integer> ages = new HashMap<>();
```

`Map<K, V>`의 타입 매개변수는 다음을 의미한다.

|타입|의미|예시|
|---|---|---|
|`K`|키의 타입|`String`|
|`V`|값의 타입|`Integer`|

위 Map은 문자열을 키로, 정수를 값으로 사용한다.

```
ages.put("철수", 25);
ages.put("영희", 28);
```

제네릭에는 기본형인 `int`, `long` 등을 직접 사용할 수 없으므로 래퍼 클래스를 사용한다.

```
Map<String, Integer> map = new HashMap<>(); // 가능

// Map<String, int> map = new HashMap<>();  // 컴파일 오류
```

`25` 같은 `int` 값을 넣을 때는 오토박싱으로 `Integer`로 변환된다.

---

## 기본 메서드

### 저장 및 수정: put()

```
Map<String, Integer> scores = new HashMap<>();

scores.put("철수", 80);

Integer previous = scores.put("철수", 95);

System.out.println(previous);           // 80
System.out.println(scores.get("철수")); // 95
```

`put()`은 **기존에 연결되어 있던 값**을 반환한다. 기존 매핑이 없었다면 `null`을 반환한다.

단, 기존 값이 `null`이었던 경우에도 반환값은 `null`이다.

### 조회: get()

```
System.out.println(scores.get("철수")); // 95
System.out.println(scores.get("민수")); // null
```

키가 없으면 `null`을 반환한다.

이때 기본형으로 바로 받으면 주의해야 한다.

```
int score = scores.get("민수");
// null을 int로 언박싱하면서 NullPointerException 발생
```

값이 없을 수 있다면 `Integer`로 받거나 기본값을 지정한다.

```
Integer nullableScore = scores.get("민수");
int defaultScore = scores.getOrDefault("민수", 0);
```

### 키가 없을 때 기본값 반환: getOrDefault()

```
Map<String, Integer> stock = new HashMap<>();

stock.put("apple", 10);

System.out.println(stock.getOrDefault("apple", 0));  // 10
System.out.println(stock.getOrDefault("banana", 0)); // 0
```

**기본값을 반환할 뿐, Map에 해당 키를 추가하지는 않는다.**

또한 키가 존재하고 그 값이 `null`이면 기본값이 아니라 `null`을 반환한다.

```
stock.put("banana", null);

System.out.println(stock.getOrDefault("banana", 0)); // null
```

### 존재 여부 확인: containsKey(), containsValue()

```
System.out.println(stock.containsKey("apple")); // true
System.out.println(stock.containsValue(10));    // true
```

`get(key) == null`만으로는 다음 두 상황을 구분할 수 없다.

1. 키가 존재하지 않는다.
2. 키는 존재하지만 값이 `null`이다.

둘을 구분해야 한다면 `containsKey()`를 사용한다.

### 삭제: remove()

```
Map<String, Integer> scores = new HashMap<>();

scores.put("철수", 95);

Integer removed = scores.remove("철수");

System.out.println(removed); // 95
```

키와 값이 모두 일치할 때만 삭제할 수도 있다.

```
scores.put("영희", 90);

boolean first = scores.remove("영희", 80);  // false
boolean second = scores.remove("영희", 90); // true
```

### 크기 확인 및 전체 삭제

```
scores.size();    // 저장된 키-값 쌍의 개수
scores.isEmpty(); // 비어 있는지 확인
scores.clear();   // 모든 매핑 삭제
```

주요 메서드를 정리하면 다음과 같다.

|메서드|역할|
|---|---|
|`put(key, value)`|추가 또는 교체|
|`get(key)`|값 조회|
|`getOrDefault(key, defaultValue)`|키가 없으면 기본값 반환|
|`containsKey(key)`|키 존재 여부 확인|
|`containsValue(value)`|값 존재 여부 확인|
|`remove(key)`|매핑 삭제|
|`size()`|매핑 개수 확인|
|`isEmpty()`|비어 있는지 확인|
|`clear()`|전체 삭제|

---

## Map 순회하기

### 키와 값이 모두 필요할 때: entrySet()

`Map.Entry<K, V>`는 하나의 키-값 쌍을 표현한다.

```
Map<String, Integer> scores = new HashMap<>();

scores.put("철수", 95);
scores.put("영희", 90);

for (Map.Entry<String, Integer> entry : scores.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}
```

키와 값이 모두 필요하다면 `entrySet()`을 사용하면 된다.

### 키만 필요할 때: keySet()

```
for (String name : scores.keySet()) {
    System.out.println(name);
}
```

### 값만 필요할 때: values()

```
for (Integer score : scores.values()) {
    System.out.println(score);
}
```

`values()`는 `Set`이 아니라 `Collection`을 반환한다. 값은 중복될 수 있기 때문이다.

### 람다식 사용: forEach()

Java 8 이상에서는 다음처럼 작성할 수 있다.

```
scores.forEach((name, score) -> {
    System.out.println(name + ": " + score);
});
```

**순회 순서는 사용하는 구현체에 따라 달라진다.** `HashMap`의 출력 순서를 삽입 순서라고 가정하면 안 된다.

---

## 주요 구현체 비교

|구현체|순서|`null` 키|`null` 값|동시 접근|
|---|---|---|---|---|
|`HashMap`|보장 없음|하나 허용|허용|별도 동기화 필요|
|`LinkedHashMap`|기본적으로 삽입 순서|하나 허용|허용|별도 동기화 필요|
|`TreeMap`|키 정렬 순서|자연 정렬에서는 불가¹|허용|별도 동기화 필요|
|`ConcurrentHashMap`|보장 없음|불가|불가|동시 읽기·수정 지원|

`null`을 처리하도록 만든 `Comparator`를 제공하면 허용할 수 있다.

### HashMap: 일반적인 기본 선택

키로 값을 저장하고 조회하는 일반적인 상황에서 사용한다.

```
Map<String, Integer> map = new HashMap<>();

map.put("banana", 2);
map.put("apple", 1);
map.put("cherry", 3);
```

삽입한 순서나 키의 정렬 순서를 보장하지 않는다.

해시가 적절히 분산된다는 전제에서 기본적인 조회·추가·삭제는 평균적으로 `O(1)`이다.

### LinkedHashMap: 삽입 순서 유지

```
Map<String, Integer> map = new LinkedHashMap<>();

map.put("banana", 2);
map.put("apple", 1);
map.put("cherry", 3);

System.out.println(map);
// {banana=2, apple=1, cherry=3}
```

기본 설정에서는 먼저 넣은 키부터 순회한다. 기존 키의 값을 바꿔도 그 키의 순서는 유지된다.

생성자 옵션을 사용하면 **접근 순서**로 관리할 수도 있어 캐시 구현에 활용할 수 있다.

### TreeMap: 키 정렬

```
Map<String, Integer> map = new TreeMap<>();

map.put("banana", 2);
map.put("apple", 1);
map.put("cherry", 3);

System.out.println(map);
// {apple=1, banana=2, cherry=3}
```

기본적으로 키의 자연 순서로 정렬한다. 문자열이라면 문자열 비교 순서를 따른다.

역순 정렬도 가능하다.

```
Map<String, Integer> map =
        new TreeMap<>(Comparator.reverseOrder());
```

주요 조회·추가·삭제 연산은 `O(log n)`이다. 범위 조회나 특정 키의 앞뒤 키를 찾는 작업에도 유용하다.

**값이 아닌 키를 기준으로 정렬한다는 점**이 중요하다.

### ConcurrentHashMap: 여러 스레드에서 공유

```
Map<String, Integer> counts = new ConcurrentHashMap<>();

counts.merge("apple", 1, Integer::sum);
```

여러 스레드가 동시에 읽고 수정할 수 있도록 설계된 구현체이다.

다만 개별 메서드가 안전하다고 해서 여러 메서드를 조합한 코드까지 하나의 원자적 작업이 되는 것은 아니다.

```
// 조회와 저장 사이에 다른 스레드가 값을 변경할 수 있음
counts.put("apple", counts.getOrDefault("apple", 0) + 1);
```

이런 카운터 갱신에는 `ConcurrentHashMap`의 `merge()`처럼 원자적으로 처리되는 연산을 사용한다.

---

## HashMap의 동작 원리: hashCode()와 equals()

`HashMap`은 키의 해시값을 이용해 데이터를 저장하고 찾는다.

개념적으로는 다음 순서로 동작한다.

1. 키의 `hashCode()`를 이용해 저장 위치인 **버킷(bucket)**을 결정한다.
2. 해당 버킷에서 후보 키를 찾는다.
3. 키가 같은 객체이거나 `equals()`로 같다고 판단되면 같은 키로 처리한다.

### 해시값이 같다고 같은 키는 아니다

서로 다른 키가 같은 해시값이나 버킷을 갖는 상황을 **해시 충돌**이라고 한다.

`HashMap`은 충돌이 발생해도 키를 비교해서 구분한다.

여기서 중요한 규칙은 다음과 같다.

> `equals()`가 `true`인 두 객체는 반드시 같은 `hashCode()`를 반환해야 한다.

반대로 해시값이 같다고 해서 반드시 `equals()`가 `true`일 필요는 없다.

### 사용자 정의 객체를 키로 사용할 때

직접 만든 클래스를 논리적인 값 기준으로 비교하려면 `equals()`와 `hashCode()`를 함께 구현해야 한다.

```
import java.util.Objects;

final class UserKey {
    private final long id;

    UserKey(long id) {
        this.id = id;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) {
            return true;
        }

        if (!(obj instanceof UserKey)) {
            return false;
        }

        UserKey other = (UserKey) obj;
        return id == other.id;
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

이제 서로 다른 객체라도 `id`가 같으면 동일한 키로 취급된다.

```
Map<UserKey, String> users = new HashMap<>();

users.put(new UserKey(1L), "철수");

System.out.println(users.get(new UserKey(1L))); // 철수
```

### 키의 비교 기준이 되는 필드는 바꾸지 않는 것이 좋다

키를 넣은 후 `hashCode()`나 `equals()`에 영향을 주는 필드를 바꾸면, 저장된 위치와 조회할 때 계산한 위치가 달라져 조회에 실패할 수 있다.

따라서 `String`, `Integer` 또는 불변 객체를 키로 사용하는 것이 좋다.

참고로 `TreeMap`은 해시 대신 비교 결과를 사용한다. `compareTo()` 또는 `Comparator.compare()`의 결과가 `0`이면 같은 키로 취급하므로, 비교 기준을 `equals()`와 일관되게 설계해야 한다.

---

## 유용한 갱신 메서드

### putIfAbsent(): 값이 없을 때 저장

```
Map<String, Integer> scores = new HashMap<>();

scores.putIfAbsent("철수", 80);
scores.putIfAbsent("철수", 95);

System.out.println(scores.get("철수")); // 80
```

키가 없거나 기존 값이 `null`이면 새 값을 넣는다.

### computeIfAbsent(): 필요한 경우에만 값 생성

그룹별 목록을 만들 때 자주 사용한다.

```
Map<String, List<String>> teams = new HashMap<>();

teams.computeIfAbsent("개발팀", key -> new ArrayList<>())
     .add("철수");

teams.computeIfAbsent("개발팀", key -> new ArrayList<>())
     .add("영희");

System.out.println(teams);
// {개발팀=[철수, 영희]}
```

동작은 다음과 같다.

1. `"개발팀"`에 연결된 값이 없거나 `null`이면 새 리스트를 만든다.
2. 만든 리스트를 Map에 저장한다.
3. 기존 리스트 또는 새 리스트를 반환한다.
4. 반환받은 리스트에 이름을 추가한다.

계산 함수가 `null`을 반환하면 새 값을 저장하지 않는다.

### merge(): 개수나 합계 누적

단어별 등장 횟수를 세는 예시이다.

```
String[] words = {"java", "spring", "java", "map", "spring", "java"};

Map<String, Integer> counts = new HashMap<>();

for (String word : words) {
    counts.merge(word, 1, Integer::sum);
}

System.out.println(counts.get("java"));   // 3
System.out.println(counts.get("spring")); // 2
System.out.println(counts.get("map"));    // 1
```

```
counts.merge(word, 1, Integer::sum);
```

이 코드는 다음 의미이다.

- 기존 값이 없거나 `null`이면 `1`을 저장한다.
- 기존 값이 있으면 기존 값과 `1`을 더해 저장한다.

병합 함수가 `null`을 반환하면 해당 매핑을 삭제한다. `merge()`에 전달하는 새 값 자체는 `null`일 수 없다.

---

## Map을 사용할 때 자주 하는 실수

### 일반적인 순회 도중 직접 삭제하기

다음 코드는 `HashMap`에서 `ConcurrentModificationException`을 일으킬 수 있다.

```
for (String key : map.keySet()) {
    if (key.startsWith("temp")) {
        map.remove(key);
    }
}
```

조건에 맞는 항목을 삭제하려면 `removeIf()`를 사용할 수 있다.

```
map.entrySet().removeIf(
        entry -> entry.getKey().startsWith("temp")
);
```

이 예시는 키가 `null`이 아니며 수정 가능한 Map이라는 전제이다.

### keySet(), values(), entrySet()을 복사본으로 생각하기

이 메서드들은 일반적으로 원본 Map과 연결된 **뷰(view)**를 반환한다.

```
Map<String, Integer> scores = new HashMap<>();
scores.put("철수", 90);

Set<String> keys = scores.keySet();
keys.remove("철수");

System.out.println(scores.isEmpty()); // true
```

독립적인 복사본이 필요하면 따로 생성해야 한다.

```
Set<String> copiedKeys = new HashSet<>(scores.keySet());
```

### Map에 넣은 객체가 자동으로 복사된다고 생각하기

Map은 객체의 참조를 저장한다.

```
List<String> members = new ArrayList<>();
Map<String, List<String>> teams = new HashMap<>();

teams.put("개발팀", members);

members.add("철수");

System.out.println(teams.get("개발팀")); // [철수]
```

Map에 넣은 리스트와 `members`는 같은 객체를 가리킨다.

`new HashMap<>(original)`로 Map을 복사해도 내부 키와 값 객체까지 복제하지는 않는다. 이는 **얕은 복사**이다.

### 동시성 Map이면 내부 값도 안전하다고 생각하기

```
Map<String, List<String>> teams = new ConcurrentHashMap<>();
```

이렇게 선언해도 값으로 저장된 `ArrayList`까지 스레드 안전해지는 것은 아니다.

Map의 동시 접근과 내부 객체의 동시 수정은 각각 고려해야 한다.

---

## 수정할 수 없는 Map 만들기

Java 9 이상에서는 `Map.of()`로 간단하게 만들 수 있다.

```
Map<String, Integer> codes = Map.of(
        "OK", 200,
        "NOT_FOUND", 404,
        "SERVER_ERROR", 500
);
```

이 Map은 매핑을 추가·삭제·교체할 수 없다.

```
codes.put("CREATED", 201);
// UnsupportedOperationException
```

주요 특징은 다음과 같다.

- `null` 키와 값을 허용하지 않는다.
- 중복 키를 전달하면 예외가 발생한다.
- 순회 순서를 보장하지 않는다.
- 내부 값 객체의 변경까지 막는 것은 아니다.

기존 Map으로부터 수정 불가능한 Map을 만들려면 Java 10 이상의 `Map.copyOf()`를 사용할 수 있다.

```
Map<String, Integer> original = new HashMap<>();
original.put("철수", 90);

Map<String, Integer> copy = Map.copyOf(original);
```

이후 `original`의 매핑을 바꿔도 `copy`의 매핑은 바뀌지 않는다. 단, 키와 값 객체는 깊게 복사되지 않으며, 원본에 `null` 키나 값이 있으면 예외가 발생한다.

---

## 각 Map의 적합한 상황

| 필요한 기능             | 적합한 상황              |
| ------------------ | ------------------- |
| 일반적인 키 기반 저장과 조회   | `HashMap`           |
| 삽입한 순서대로 순회        | `LinkedHashMap`     |
| 키 정렬, 범위 조회        | `TreeMap`           |
| 여러 스레드에서 공유하며 수정   | `ConcurrentHashMap` |
| 작은 고정 매핑 생성        | `Map.of()`          |
| 기존 매핑의 수정 불가능한 복사본 | `Map.copyOf()`      |
