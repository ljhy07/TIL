# ConcurrentHashMap

`ConcurrentHashMap`은 여러 스레드가 동시에 접근하는 상황에서 안전하게 사용할 수 있도록 만들어진 Java의 `Map` 구현체이다.

한 문장 요약

> 여러 스레드가 동시에 데이터를 읽고 수정할 수 있다면 `HashMap` 대신 고려해야 하는 스레드 안전한 Map이다.

다만 `ConcurrentHashMap`을 사용한다고 해서 그 주변의 모든 로직까지 자동으로 스레드 안전해지는 것은 아니다.

---

## HashMap을 그대로 사용하지 않는 이유

일반적인 `HashMap`은 스레드 안전하지 않다.

```
Map<String, Integer> map = new HashMap<>();

map.put("apple", 1);
Integer count = map.get("apple");
```

하나의 스레드만 접근한다면 문제가 없다. 하지만 여러 스레드가 동시에 아래 작업을 수행하면 문제가 생길 수 있다.

```
map.put("apple", 1);
map.remove("apple");
map.get("apple");
```

발생 가능한 문제는 다음과 같다.

- 어떤 스레드의 수정 결과가 다른 스레드에 제대로 보이지 않을 수 있음
- 동시에 수정하면서 데이터가 유실될 수 있음
- 순회 도중 수정하면 `ConcurrentModificationException`이 발생할 수 있음
- 여러 연산을 조합한 코드에서 경쟁 상태가 발생할 수 있음

이때 사용할 수 있는 대표적인 자료구조가 `ConcurrentHashMap`이다.

```
Map<String, Integer> map = new ConcurrentHashMap<>();
```

---

## 기본 사용법

```
import java.util.concurrent.ConcurrentHashMap;

ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

map.put("apple", 10);
map.put("banana", 20);

Integer appleCount = map.get("apple");

map.remove("banana");
```

가능하면 변수 타입은 인터페이스인 `Map`으로 선언하는 것도 좋다.

```
Map<String, Integer> map = new ConcurrentHashMap<>();
```

다만 `ConcurrentHashMap` 고유 메서드를 명확하게 사용하려는 코드에서는 구현체 타입으로 선언해도 괜찮다.

---

## ConcurrentHashMap이 보장하는 것

`ConcurrentHashMap`의 주요 특징은 다음과 같다.

- 여러 스레드가 동시에 `get()`을 호출할 수 있음
- 여러 스레드가 동시에 데이터를 추가하거나 수정할 수 있음
- 읽기와 쓰기가 동시에 일어나도 내부 자료구조가 깨지지 않음
- 일반적으로 전체 Map 하나를 잠그지 않기 때문에 동시 처리 성능이 좋음
- 순회 중 다른 스레드가 데이터를 수정해도 일반적으로 `ConcurrentModificationException`이 발생하지 않음

예를 들어 다음 코드는 여러 스레드에서 호출되어도 Map 자체는 안전하다.

```
private final Map<String, String> cache = new ConcurrentHashMap<>();

public void save(String key, String value) {
    cache.put(key, value);
}

public String find(String key) {
    return cache.get(key);
}
```

여기서 말하는 “안전하다”는 것은 `ConcurrentHashMap` 내부의 데이터 구조가 망가지지 않는다는 뜻이다.

---

## 각각의 메서드가 안전하다고 전체 로직도 안전한 것은 아니다

`get()`, `put()`, `remove()` 같은 각각의 메서드는 스레드 안전하다.

하지만 여러 메서드를 조합한 로직은 자동으로 원자적이지 않다.

### 잘못된 예: 조회한 다음 추가하기

```
if (!map.containsKey("apple")) {
    map.put("apple", 1);
}
```

두 스레드가 동시에 실행하면 다음 상황이 가능하다.

1. 스레드 A가 `containsKey()`를 실행하고 값이 없음을 확인
2. 스레드 B도 값이 없음을 확인
3. 스레드 A가 값을 추가
4. 스레드 B도 값을 추가

각각의 메서드는 안전하지만 `확인 → 추가`라는 전체 동작은 하나의 원자적인 작업이 아니다.

### 올바른 방법: putIfAbsent()

```
map.putIfAbsent("apple", 1);
```

`putIfAbsent()`는 다음 작업을 하나의 원자적인 연산으로 처리한다.

> 해당 키가 없으면 값을 추가하고, 이미 있으면 기존 값을 유지한다.

반환값도 알아두면 좋다.

```
Integer previousValue = map.putIfAbsent("apple", 1);

if (previousValue == null) {
    System.out.println("새 값이 추가됨");
} else {
    System.out.println("이미 존재하는 값: " + previousValue);
}
```

---

## 카운터 증가에는 get() + put()을 사용하면 안 된다

다음 코드는 겉보기에는 정상적이지만 동시성 문제가 있다.

```
Integer count = map.get("apple");
map.put("apple", count + 1);
```

두 스레드가 동시에 실행하면 증가 결과가 유실될 수 있다.

예를 들어 기존 값이 `10`일 때:

1. 스레드 A가 10을 읽음
2. 스레드 B도 10을 읽음
3. 스레드 A가 11을 저장
4. 스레드 B도 11을 저장

두 번 증가했지만 최종 결과는 `12`가 아니라 `11`이다.

이것을 갱신 유실, 즉 lost update라고 한다.

### 방법 1: compute()

```
map.compute("apple", (key, value) -> {
    if (value == null) {
        return 1;
    }

    return value + 1;
});
```

간단하게 작성하면:

```
map.compute("apple", (key, value) -> value == null ? 1 : value + 1);
```

### 방법 2: merge()

카운터 증가에는 `merge()`가 더 읽기 좋을 때가 많다.

```
map.merge("apple", 1, Integer::sum);
```

의미는 다음과 같다.

- `"apple"`이 없으면 `1`을 저장
- 이미 값이 있으면 기존 값과 `1`을 더해서 저장

단어 개수를 세는 코드에도 자주 사용한다.

```
Map<String, Integer> wordCounts = new ConcurrentHashMap<>();

wordCounts.merge("java", 1, Integer::sum);
wordCounts.merge("java", 1, Integer::sum);
wordCounts.merge("spring", 1, Integer::sum);
```

결과:

```
java   = 2
spring = 1
```

---

## computeIfAbsent()

키가 없을 때만 값을 생성하고 추가하고 싶다면 `computeIfAbsent()`를 사용한다.

### 좋지 않은 예

```
if (!map.containsKey(key)) {
    map.put(key, loadData(key));
}

return map.get(key);
```

`containsKey()`와 `put()` 사이에 다른 스레드가 개입할 수 있다.

### 올바른 예

```
return map.computeIfAbsent(key, this::loadData);
```

예를 들어 사용자 정보를 캐시한다면:

```
private final Map<Long, User> userCache = new ConcurrentHashMap<>();

public User getUser(Long userId) {
    return userCache.computeIfAbsent(
        userId,
        id -> userRepository.findById(id)
    );
}
```

다만 계산 함수는 가능하면 짧고 단순해야 한다.

```
map.computeIfAbsent(key, k -> {
    // 너무 오래 걸리는 외부 API 호출
    // 복잡한 Map 수정
    // 재귀적으로 같은 키 수정
    return createValue(k);
});
```

계산이 진행되는 동안 같은 키를 수정하려는 다른 작업이 기다릴 수 있기 때문이다.

특히 계산 함수 내부에서 같은 Map의 같은 키를 다시 수정하는 코드는 피해야 한다.

```
map.computeIfAbsent("apple", key -> {
    map.put("apple", 10); // 피해야 하는 코드
    return 20;
});
```

추가로 계산 함수가 `null`을 반환하면 값이 저장되지 않는다.

```
map.computeIfAbsent("apple", key -> null);

System.out.println(map.containsKey("apple")); // false
```

---

## null 키와 null 값을 허용하지 않는다

`ConcurrentHashMap`에는 `null` 키와 `null` 값을 넣을 수 없다.

```
Map<String, String> map = new ConcurrentHashMap<>();

map.put(null, "value"); // NullPointerException
map.put("key", null);   // NullPointerException
```

왜냐하면 동시성 환경에서는 `get()`이 `null`을 반환했을 때 의미가 모호해질 수 있기 때문이다.

```
String value = map.get("key");
```

`value == null`이면 다음 중 무엇인지 구분하기 어려울 수 있다.

- 실제로 키가 없음
- 키는 있지만 값이 `null`임
- 다른 스레드가 방금 제거함

`ConcurrentHashMap`은 값으로 `null`을 금지하여 적어도 “조회 결과가 `null`이면 현재 조회 시점에는 값이 없었다”는 의미를 유지한다.

값이 없다는 상태를 표현해야 한다면 보통 다음 방법을 고려한다.

- 해당 키 자체를 저장하지 않음
- 별도의 상태 객체 사용
- 상황에 따라 `Optional` 사용

다만 `Map<K, Optional<V>>`는 설계를 복잡하게 만들 수 있으므로 꼭 필요한 경우에만 사용하는 것이 좋다.

---

## 순회 중 수정할 수 있지만 정확한 한 시점의 스냅샷은 아니다

`ConcurrentHashMap`은 다른 스레드가 수정하는 동안에도 순회할 수 있다.

```
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}
```

다른 스레드가 동시에 값을 추가하거나 제거해도 일반적으로 `ConcurrentModificationException`이 발생하지 않는다.

하지만 순회 결과가 특정 순간의 정확한 전체 상태를 보장하는 것은 아니다.

순회 도중 일어난 변경은:

- 보일 수도 있음
- 보이지 않을 수도 있음
- 일부 변경만 보일 수도 있음

이를 흔히 약한 일관성, 즉 weak consistency라고 부른다.

따라서 다음과 같이 생각해야 한다.

> 순회는 안전하지만, 모든 데이터가 정확히 같은 시점의 상태라고 가정해서는 안 된다.

정확한 스냅샷이 필요하다면 복사본을 만들 수 있다.

```
Map<String, Integer> snapshot = new HashMap<>(map);
```

다만 복사하는 도중에도 수정이 계속되면 이 복사본 역시 엄밀한 의미에서 하나의 순간을 완벽하게 나타내지는 않을 수 있다. 정말 정확한 스냅샷이 필요하면 별도의 잠금이나 설계가 필요합니다. 이는 주니어 단계에서는 “추가적인 동기화가 필요할 수 있다” 정도로 이해하면 충분하다.

---

## size(), isEmpty(), containsValue()를 동시성 제어에 사용이 안된다

다른 스레드가 계속 수정하는 상황에서 다음 코드는 위험할 수 있다.

```
if (map.size() < 100) {
    map.put(key, value);
}
```

`size()`를 확인한 직후 다른 스레드가 데이터를 추가할 수 있다. 따라서 최종 크기는 100을 초과할 수 있다.

마찬가지로:

```
if (!map.isEmpty()) {
    process(map);
}
```

`isEmpty()` 확인 후 다른 스레드가 모든 데이터를 제거할 수 있다.

`size()`, `isEmpty()`, `containsValue()`는 상태를 관찰하거나 모니터링하는 용도로는 사용할 수 있지만, 그 결과를 기반으로 중요한 동시성 제어를 해서는 안 된다.

```
log.info("현재 캐시 데이터 개수: {}", map.size());
```

이런 용도는 괜찮다.

---

## ConcurrentHashMap에 들어 있는 객체까지 스레드 안전한 것은 아니다

이 부분은 매우 중요하다.

```
Map<String, ArrayList<String>> map = new ConcurrentHashMap<>();

map.put("users", new ArrayList<>());
```

`ConcurrentHashMap`은 키와 값의 저장 및 조회를 안전하게 해 준다. 하지만 값으로 들어간 `ArrayList`까지 스레드 안전하게 만들어 주지는 않는다.

다음 코드는 문제가 될 수 있다.

```
map.get("users").add("Alice");
```

여러 스레드가 같은 `ArrayList`에 동시에 `add()`하면 문제가 발생할 수 있다.

필요하다면 값으로도 스레드 안전한 자료구조를 사용해야 한다.

```
Map<String, CopyOnWriteArrayList<String>> map =
    new ConcurrentHashMap<>();

map.computeIfAbsent(
    "users",
    key -> new CopyOnWriteArrayList<>()
).add("Alice");
```

또는 목적에 맞는 자료구조를 선택해야 한다.

```
Map<String, ConcurrentLinkedQueue<String>> map =
    new ConcurrentHashMap<>();
```

핵심은 다음과 같다.

> ConcurrentHashMap은 Map에 대한 접근을 보호할 뿐, Map 안에 들어 있는 가변 객체의 내부 변경까지 보호하지 않는다.

---

## 자주 사용하는 원자적 메서드

### putIfAbsent()

키가 없을 때만 추가한다.

```
map.putIfAbsent("apple", 10);
```

### remove(key, value)

키와 값이 모두 일치할 때만 제거한다.

```
map.remove("apple", 10);
```

다른 스레드가 값을 이미 변경했다면 제거되지 않는다.

### replace(key, oldValue, newValue)

현재 값이 예상한 값일 때만 변경한다.

```
boolean replaced = map.replace("apple", 10, 20);
```

의미:

> `"apple"`의 현재 값이 10일 때만 20으로 바꾼다.

### computeIfAbsent()

키가 없을 때 값을 계산해서 추가한다.

```
map.computeIfAbsent(key, this::createValue);
```

### computeIfPresent()

키가 있을 때만 값을 계산해서 변경한다.

```
map.computeIfPresent("apple", (key, value) -> value + 1);
```

### compute()

키의 존재 여부와 관계없이 값을 계산합니다.

```
map.compute("apple", (key, value) ->
    value == null ? 1 : value + 1
);
```

### merge()

키가 없으면 값을 넣고, 있으면 기존 값과 합친다.

```
map.merge("apple", 1, Integer::sum);
```

---

## remove()도 조합해서 쓰면 경쟁 상태가 생길 수 있다

다음 코드는 의도와 다르게 동작할 수 있다.

```
if (map.get("apple") == 10) {
    map.remove("apple");
}
```

값을 확인한 뒤 `remove()`를 실행하기 전에 다른 스레드가 값을 20으로 바꿀 수 있다. 그러면 현재 값이 20인데도 삭제될 수 있다.

이때는 조건부 삭제 메서드를 사용한다.

```
map.remove("apple", 10);
```

현재 값이 정확히 10일 때만 삭제된다.

값 비교에는 `==`보다 `equals()` 또는 조건부 메서드를 사용하는 것이 좋다는 점도 함께 기억해야 한다.

---

## Collections.synchronizedMap과의 차이

스레드 안전한 Map을 만드는 또 다른 방법도 있다.

```
Map<String, Integer> map =
    Collections.synchronizedMap(new HashMap<>());
```

이 방식은 일반적으로 Map에 접근할 때 하나의 잠금을 사용하는 방식에 가깝다. 여러 스레드가 동시에 많이 접근하면 대기하는 시간이 커질 수 있다.

또한 순회할 때는 직접 동기화해야 한다.

```
synchronized (map) {
    for (Map.Entry<String, Integer> entry : map.entrySet()) {
        System.out.println(entry);
    }
}
```

반면 `ConcurrentHashMap`은 동시 접근을 더 적극적으로 허용하도록 설계되어 있고, 순회할 때 직접 `synchronized` 블록을 작성하지 않아도 된다.

실무에서 여러 스레드가 자주 읽고 쓰는 Map이라면 보통 `ConcurrentHashMap`을 먼저 고려한다.

---

## ConcurrentHashMap을 사용해야 하는 경우

다음과 같은 상황에서 자주 사용한다.

### 여러 스레드가 공유하는 캐시

```
private final Map<Long, User> cache = new ConcurrentHashMap<>();
```

### 사용자별 상태 관리

```
private final Map<Long, Session> sessions = new ConcurrentHashMap<>();
```

### 데이터별 카운트

```
private final Map<String, Integer> counts = new ConcurrentHashMap<>();
```

### 작업 진행 상태 저장

```
private final Map<String, JobStatus> jobStatuses =
    new ConcurrentHashMap<>();
```

### 키별 객체를 한 번만 생성

```
private final Map<String, Client> clients =
    new ConcurrentHashMap<>();

public Client getClient(String name) {
    return clients.computeIfAbsent(name, Client::new);
}
```

---

## ConcurrentHashMap을 사용하지 않아도 되는 경우

다음 상황에서는 일반 `HashMap`이 더 적절할 수 있다.

- 하나의 스레드에서만 사용하는 지역 변수
- 메서드 안에서 생성하고 외부에 공유하지 않는 Map
- 생성 후 수정하지 않는 읽기 전용 Map
- 이미 외부 잠금으로 모든 접근이 보호되는 경우
- Map 전체에 걸친 복잡한 작업을 하나의 원자적 작업으로 처리해야 하는 경우

예를 들어 다음 Map은 메서드 내부에서만 사용되므로 `ConcurrentHashMap`이 필요 없다.

```
public Map<String, Integer> calculate() {
    Map<String, Integer> result = new HashMap<>();

    result.put("apple", 10);
    result.put("banana", 20);

    return result;
}
```

`ConcurrentHashMap`은 “혹시 모르니 무조건 사용하는 Map”이 아니다. 실제로 여러 스레드가 공유할 때 사용하는 것이 좋다.

---

## synchronized와 함께 써야 할 때도 있다

`ConcurrentHashMap`은 개별 키에 대한 작업이나 제공되는 원자적 메서드에는 강다. 하지만 여러 키를 하나의 단위로 변경하는 작업까지 자동으로 보장하지 않는다.

```
map.remove("accountA");
map.put("accountB", value);
```

두 작업 사이에 다른 스레드가 Map을 볼 수 있다.

만약 두 작업이 반드시 하나처럼 처리되어야 한다면 별도의 잠금이 필요할 수 있다.

```
private final Object lock = new Object();

synchronized (lock) {
    map.remove("accountA");
    map.put("accountB", value);
}
```

다만 여러 키를 동시에 원자적으로 처리해야 하는 요구가 많다면 단순히 `ConcurrentHashMap`을 추가하는 것만으로 해결하지 말고, 데이터 구조나 전체 설계를 다시 확인하는 것이 좋다.

주니어 단계에서는 다음 정도로 기억하면 충분하다.

> 하나의 ConcurrentHashMap 메서드는 안전하지만, 여러 메서드 또는 여러 키에 걸친 비즈니스 로직은 별도 보호가 필요할 수 있다.

---

## 실무에서 자주 하는 실수

### 실수 1: containsKey() 후 put()

```
if (!map.containsKey(key)) {
    map.put(key, value);
}
```

수정:

```
map.putIfAbsent(key, value);
```

또는:

```
map.computeIfAbsent(key, this::createValue);
```

### 실수 2: get() 후 증가

```
map.put(key, map.get(key) + 1);
```

수정:

```
map.merge(key, 1, Integer::sum);
```

### 실수 3: Map이 안전하니 값도 안전하다고 생각함

```
Map<String, ArrayList<String>> map =
    new ConcurrentHashMap<>();
```

`ArrayList`는 여전히 스레드 안전하지 않다.

### 실수 4: size()를 기준으로 정확한 제어

```
if (map.size() < limit) {
    map.put(key, value);
}
```

확인과 추가 사이에 다른 스레드가 개입할 수 있다.

### 실수 5: null 저장

```
map.put(key, null);
```

`NullPointerException`이 발생한다.

### 실수 6: compute 함수 안에서 오래 걸리는 작업 수행

```
map.computeIfAbsent(key, k -> slowExternalApiCall(k));
```

기능상 가능하더라도 같은 키를 기다리는 스레드가 오래 대기할 수 있다. 캐시 로딩에 적합한지, 실패와 재시도를 어떻게 처리할지 함께 생각해야 한다.