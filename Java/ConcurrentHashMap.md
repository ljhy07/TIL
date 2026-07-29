## 한 문장으로 정의

`ConcurrentHashMap<K, V>`은 여러 스레드가 동시에 읽고 수정할 수 있도록 설계된 Java의 스레드 안전한 해시 맵이다.

핵심
- `get()` 같은 조회는 일반적으로 잠금 없이 진행된다.
- 수정 작업은 맵 전체가 아니라 충돌이 발생한 일부 영역을 중심으로 동기화된다.

따라서 모든 작업을 하나의 락으로 직렬화하는 `Hashtable`이나 `Collections.synchronizedMap()`보다 높은 동시성을 기대할 수 있다.

---

## 필요 여부

일반 `HashMap`은 동시 수정에 안전하지 않는다.

```
Map<String, Integer> map = new HashMap<>();

// 여러 스레드가 동시에 실행
map.put("A", 1);
map.put("B", 2);
```

이 경우 다음 문제가 생길 수 있다.

- 값 유실
- 내부 구조 손상
- 다른 스레드에서 최신 값이 보이지 않는 메모리 가시성 문제
- 반복 도중 `ConcurrentModificationException`
- 읽는 쪽에서도 예측하기 어려운 결과

단순하게 맵 전체를 `synchronized`로 감쌀 수도 있다.

```
Map<String, Integer> map =
        Collections.synchronizedMap(new HashMap<>());
```

그러나 이 방식은 보통 하나의 락으로 맵 접근을 직렬화한다. 한 스레드가 수정하는 동안 관계없는 키를 처리하는 다른 스레드도 대기할 수 있다.

`ConcurrentHashMap`은 이 병목을 줄이는 데 초점을 둔다.

```
ConcurrentHashMap<String, Integer> map =
        new ConcurrentHashMap<>();
```

---

## 다른 Map과의 차이

| 구현체                 | 스레드 안전 | `null` 허용    | 동시성 특성                       | 반복자               |
| ------------------- | ------ | ------------ | ---------------------------- | ----------------- |
| `HashMap`           | 아니요    | 키 1개, 값 여러 개 | 동시 접근에 부적합                   | fail-fast         |
| `Hashtable`         | 예      | 아니요          | 대부분 전체 객체 동기화                | 비교적 낮은 동시성        |
| `synchronizedMap`   | 예      | 원본 Map에 따름   | 보통 전체 맵 락                    | 반복 시 외부 동기화 필요    |
| `ConcurrentHashMap` | 예      | 키와 값 모두 불가   | 조회와 서로 다른 영역의 수정이 높은 수준으로 병행 | weakly consistent |

`ConcurrentHashMap`은 `null` 키와 값을 허용하지 않는다.

```
map.put(null, "value"); // NullPointerException
map.put("key", null);   // NullPointerException
```

이 제약 덕분에 다음 결과를 명확하게 해석할 수 있다.

```
V value = map.get(key);

if (value == null) {
    // 실제로 매핑이 존재하지 않음
}
```

---

## 스레드 안전

각각의 단일 연산은 스레드 안전하다.

```
map.get(key);
map.put(key, value);
map.remove(key);
map.putIfAbsent(key, value);
```

하지만 여러 단일 연산을 조합한 코드까지 자동으로 원자적이 되는 것은 아니다.

### 잘못된 예: 확인 후 삽입

```
if (!map.containsKey(key)) {
    map.put(key, value);
}
```

두 스레드가 동시에 `containsKey()`에서 `false`를 확인한 다음 모두 `put()`을 실행할 수 있다.

다음처럼 복합 연산을 하나의 원자적 메서드로 표현해야 한다.

```
V previous = map.putIfAbsent(key, value);
```

### 잘못된 예: 읽고 증가시킨 후 저장

```
Integer current = map.get(key);
map.put(key, current + 1);
```

두 스레드가 모두 10을 읽은 다음 각각 11을 저장하면, 두 번 증가했지만 최종 결과는 11이 된다. 이를 lost update라고 한다.

다음과 같이 작성해야 한다.

```
map.merge(key, 1, Integer::sum);
```

또는:

```
map.compute(key, (k, oldValue) ->
        oldValue == null ? 1 : oldValue + 1);
```

즉, `ConcurrentHashMap`은 단일 연산의 안전성을 보장하지만 여러 호출 사이의 비즈니스 규칙까지 자동으로 보호하지는 않다.

---

## 주요 원자적 메서드

### `putIfAbsent`

값이 없을 때만 삽입한다.

```
User existing = users.putIfAbsent(userId, newUser);

if (existing == null) {
    // 이번 호출이 실제로 값을 등록함
}
```

다음 코드의 원자적 버전이다.

```
if (!users.containsKey(userId)) {
    users.put(userId, newUser);
}
```

### 조건부 `remove`

현재 값이 예상한 값과 일치할 때만 삭제한다.

```
boolean removed = map.remove(key, expectedValue);
```

다른 스레드가 중간에 값을 변경했다면 삭제하지 않는다.

### 조건부 `replace`

```
boolean changed =
        map.replace(key, expectedOldValue, newValue);
```

일종의 compare-and-set 형태이다.

### `computeIfAbsent`

키가 없을 때 값을 계산하여 등록한다.

```
Service service = services.computeIfAbsent(
        serviceName,
        name -> createService(name)
);
```

전체 호출은 해당 키에 대해 원자적으로 처리된다. 계산 함수가 `null`을 반환하면 아무 값도 등록되지 않는다.

### `computeIfPresent`

값이 있을 때만 갱신한다.

```
map.computeIfPresent(key, (k, oldValue) -> oldValue + 1);
```

함수가 `null`을 반환하면 해당 매핑은 삭제된다.

### `compute`

값의 존재 여부와 상관없이 계산한다.

```
map.compute(key, (k, oldValue) -> {
    if (oldValue == null) {
        return initialValue;
    }
    return update(oldValue);
});
```

결과가 `null`이면 매핑이 제거되거나 생성되지 않는다.

### `merge`

누적, 합산, 결합 작업에 특히 편리하다.

```
wordCounts.merge(word, 1, Integer::sum);
```

의미는 다음과 같다.

- 기존 값이 없으면 `1` 저장
- 기존 값이 있으면 `Integer::sum`으로 결합
- 결합 함수가 `null`을 반환하면 매핑 삭제

---

## `compute` 함수 안에서 하면 안 되는 일

`compute`, `computeIfAbsent`, `merge`에 전달하는 함수는 짧고 단순해야 한다.

```
map.computeIfAbsent(key, k -> {
    // 느린 네트워크 호출
    return remoteServer.load(k);
});
```

이 코드는 기능적으로 가능해 보이지만 계산이 진행되는 동안 다른 수정 작업이 대기할 수 있다. 같은 내부 버킷에 속한 관계없는 키에도 영향을 줄 가능성이 있다.

특히 다음 행동은 피해야 한다.

- 느린 DB 또는 네트워크 호출
- 장시간 블로킹
- 다시 같은 `ConcurrentHashMap`을 수정하는 재귀적 작업
- 여러 키를 중첩해서 수정하는 작업
- 외부 락을 복잡한 순서로 획득하는 작업

재귀적 갱신이 감지되면 `IllegalStateException`이 발생할 수도 있고, 복잡한 중첩 갱신은 데드락이나 성능 저하의 원인이 될 수 있다.

비싼 값을 한 번만 로딩해야 한다면 캐시 로딩, 중복 요청 억제, 실패 재시도 정책까지 함께 설계해야 한다. `ConcurrentHashMap` 자체는 만료 시간, 최대 크기, 자동 제거 정책을 제공하지 않는다.

---

## 내부 동작

다음 내용은 주로 현재 OpenJDK 구현에 관한 설명이며, 모든 JVM이 지켜야 하는 공개 API 계약은 아니다.

### 기본 구조

내부에는 대략 다음과 같은 테이블이 있다.

```
table
  [0] -> Node -> Node
  [1] -> null
  [2] -> TreeBin
  [3] -> Node
  ...
```

키의 `hashCode()`를 가공한 뒤 테이블 인덱스를 계산합니다. 같은 위치로 들어간 엔트리들은 하나의 bin을 구성한다.

### 조회

`get()`은 일반적으로 다음 순서로 동작한다.

1. 키의 해시 계산
2. 테이블 인덱스 결정
3. 해당 bin 탐색
4. 키가 일치하는 노드의 값 반환

조회는 일반적으로 락을 획득하지 않기 때문에 수정 작업과 동시에 실행될 수 있다.

그렇다고 “모든 상황에서 완전한 lock-free 알고리즘”이라고 이해하면 안 된다. 공개적으로 보장되는 표현은 조회 연산이 일반적으로 블로킹되지 않고 수정과 겹쳐 실행될 수 있다는 것이다.

### 삽입과 수정

비어 있는 bin에 최초 값을 넣을 때는 CAS 계열 원자 연산을 사용할 수 있다.

이미 값이 있는 bin에서는 구현상 해당 bin의 선두 노드를 중심으로 동기화하고 연결 리스트나 트리를 수정한다. 따라서 맵 전체가 아니라 충돌한 영역을 중심으로 경합이 발생한다.

```
Thread A -> bin 3 수정 ─┐
                        ├─ 서로 경합 가능
Thread B -> bin 3 수정 ─┘

Thread C -> bin 9 수정 ─── 동시에 진행 가능
```

### 충돌이 많아지면 트리로 변환

특정 bin에 충돌이 지나치게 많으면 연결 리스트가 균형 트리 구조로 바뀔 수 있다. 현대 OpenJDK 구현에서 사용되는 대표적인 내부 기준은 다음과 같다.

- 엔트리 약 8개 이상: 트리화 고려
- 테이블 크기가 64보다 작으면 트리화보다 리사이징 우선
- 엔트리가 다시 줄어들면 리스트로 되돌아갈 수 있음

이 숫자들은 구현 세부사항이므로 애플리케이션 로직이 의존해서는 안 된다.

충돌 트리는 최악의 검색 성능을 완화하지만, 좋은 `hashCode()` 구현을 대신해 주지는 않는다.

### 리사이징

엔트리가 늘어나면 더 큰 테이블을 만들어 기존 엔트리를 옮깁니다. 리사이징은 상대적으로 비싼 작업이다.

현대 구현에서는 여러 스레드가 테이블 이전 작업을 나눠 수행할 수 있다. 이전된 bin은 특별한 노드로 표시되어 다른 스레드가 새 테이블을 찾아가거나 이전 작업을 도울 수 있다.

### 크기 카운팅

매번 하나의 전역 카운터를 갱신하면 모든 수정 스레드가 같은 메모리 위치에서 경합한다. 이를 피하기 위해 구현은 기본 카운터와 분산된 카운터 셀을 사용한다. 개념적으로 `LongAdder`와 비슷한 방식이다.

---

## Java 7의 `Segment`와 Java 8 이후의 차이

오래된 자료에는 `ConcurrentHashMap`이 여러 개의 `Segment`로 분할되고 각 Segment가 별도의 락을 가진다고 설명되어 있다.

이 설명은 주로 Java 7까지의 구현이다.

Java 8 이후 구현에서는 고정된 Segment 배열 구조가 제거되었다. 대신 다음 기술을 조합한다.

- CAS
- bin 단위 동기화
- 연결 리스트와 트리
- 협력적 리사이징
- 분산 카운터

따라서 Java 8 이후 버전에서 `concurrencyLevel`을 “락 또는 Segment의 개수”로 이해하면 안 된다.

```
new ConcurrentHashMap<>(initialCapacity, loadFactor, concurrencyLevel);
```

현재 `concurrencyLevel`은 호환성을 위해 남아 있으며 초기 내부 크기 계산을 위한 힌트에 가깝다.

---

## 메모리 가시성과 happens-before

`ConcurrentHashMap`은 단순히 내부 구조가 깨지지 않도록 하는 것뿐 아니라 스레드 간 메모리 가시성도 제공한다.

특정 키에 대해 완료된 수정 작업은 그 값을 관찰한 이후의 비-null 조회와 happens-before 관계를 가진다.

예를 들어:

```
Config config = new Config(...);
configs.put("production", config);
```

다른 스레드가 다음 조회를 통해 `config`를 얻었다면:

```
Config config = configs.get("production");
```

`put()` 전에 올바르게 초기화된 `config` 상태를 안전하게 관찰할 수 있다.

하지만 맵에서 꺼낸 객체를 이후에 동기화 없이 변경해도 안전하다는 뜻은 아니다.

```
ConcurrentHashMap<String, ArrayList<String>> map =
        new ConcurrentHashMap<>();

map.computeIfAbsent("A", k -> new ArrayList<>())
   .add("value"); // ArrayList 자체는 스레드 안전하지 않음
```

`ConcurrentHashMap`이 보호하는 것은 키와 값의 매핑이다. 값 객체 내부까지 자동으로 보호하지 않는다.

필요하다면 값도 스레드 안전해야 한다.

```
ConcurrentHashMap<String, ConcurrentLinkedQueue<String>> map =
        new ConcurrentHashMap<>();

map.computeIfAbsent("A", k -> new ConcurrentLinkedQueue<>())
   .add("value");
```

또는 값의 불변 객체화, 외부 동기화, 원자적 교체 방식을 고려해야 한다.

---

## 반복자는 “약한 일관성”을 가진다

```
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry);
}
```

다른 스레드가 동시에 맵을 수정하더라도 일반적으로 `ConcurrentModificationException`이 발생하지 않는다.

다만 반복 결과는 완전한 스냅샷이 아니다.

- 반복 전에 존재했던 값이 보일 수 있음
- 반복 중 추가된 값이 보일 수도 있고 안 보일 수도 있음
- 반복 중 삭제된 값이 관찰될 수도 있음
- 맵 전체가 정확히 동일한 한 시점의 상태라고 보장되지 않음
- 순서도 보장되지 않음

이 특성을 weakly consistent iterator라고 한다.

정확한 스냅샷이 필요하다면 명시적으로 복사해야 한다.

```
Map<String, Integer> snapshot = new HashMap<>(map);
```

하지만 복사하는 동안에도 수정이 계속된다면 이 복사본 역시 하나의 완벽한 전역 시점을 나타낸다고 보장할 수는 없다. 엄밀한 스냅샷이 필요하면 별도의 락이나 버전 관리 설계가 필요하다.

---

## `size()`를 동시성 제어에 사용하면 안 된다

```
if (map.size() < 100) {
    map.put(key, value);
}
```

여러 스레드가 동시에 실행하면 모두 `size() < 100`을 확인한 뒤 삽입할 수 있으므로 크기가 100을 초과할 수 있다.

`size()`, `isEmpty()`, `containsValue()` 같은 전체 상태 메서드는 동시 수정 중에 일시적인 상태를 반영할 수 있다. 모니터링이나 추정에는 쓸 수 있지만, 동시성 제어 조건으로 사용해서는 안 된다.

`mappingCount()`는 `long`을 반환하므로 매우 큰 맵의 크기를 확인할 때 유용하지만, 동시 수정 중에는 마찬가지로 정확한 스냅샷 카운트가 아니다.

---

## 병렬 일괄 연산

`ConcurrentHashMap`은 다음 종류의 bulk operation을 제공한다.

- `forEach`
- `search`
- `reduce`

예:

```
map.forEach(
        10_000,
        (key, value) -> process(key, value)
);
```

첫 번째 인수는 `parallelismThreshold`이다.

- `Long.MAX_VALUE`: 병렬 실행을 사실상 비활성화
- `1`: 가능한 한 적극적으로 병렬화
- 중간값: 데이터 규모가 기준보다 클 때 병렬화

예를 들어 값을 합산할 수 있다.

```
long total = map.reduceValuesToLong(
        10_000,
        Integer::longValue,
        0L,
        Long::sum
);
```

주의할 점은 다음과 같다.

- 순서에 의존하면 안 됨
- 동시 수정 중이면 완전한 스냅샷 결과가 아님
- 병렬 처리 비용이 실제 작업보다 클 수 있음
- `reduce` 함수는 결합 순서에 무관하도록 연관법칙을 만족하는 것이 좋음
- 부작용이 있는 함수는 특히 주의해야 함

---

## 빈도수 집계의 권장 패턴

공식 문서에서 소개하는 대표 패턴은 `LongAdder`와 조합하는 것이다.

```
ConcurrentHashMap<String, LongAdder> frequencies =
        new ConcurrentHashMap<>();

frequencies
        .computeIfAbsent(word, key -> new LongAdder())
        .increment();
```

장점은 다음과 같다.

- 카운터 생성은 `computeIfAbsent()`로 안전하게 처리
- 동일 키의 빈번한 증가 작업은 `LongAdder`가 경합을 분산
- `Integer`를 계속 교체하는 것보다 고경합 상황에서 유리할 수 있음

현재 값은 다음처럼 읽는다.

```
long count = frequencies.get(word).sum();
```

다만 `LongAdder.sum()`도 동시 증가 중인 완벽한 원자적 스냅샷을 의미하지는 않는다.

---

## 성능 특성

일반적인 기대 시간 복잡도는 다음과 같다.

- `get()`: 평균 O(1)
- `put()`: 평균 O(1)
- `remove()`: 평균 O(1)
- 충돌 트리 탐색: 대략 O(log n)
- 전체 순회: O(n)

좋은 성능을 얻으려면 다음이 중요하다.

### 올바른 `hashCode()`

키의 해시가 특정 값에 몰리면 하나의 bin에 수정 작업이 집중된다.

```
@Override
public int hashCode() {
    return 1; // 매우 나쁜 구현
}
```

이 경우 서로 다른 키라도 같은 내부 영역에서 경합한다.

### 키는 사실상 불변이어야 함

맵에 삽입한 후 키의 `equals()`나 `hashCode()` 결과가 바뀌면 조회할 수 없게 될 수 있다.

```
MutableKey key = new MutableKey("A");
map.put(key, value);

key.setId("B"); // 위험
```

키로는 불변 객체, `String`, record 등을 사용하는 것이 안전하다.

### 초기 용량

예상 엔트리 수가 크다면 적절한 초기 용량을 주는 것이 리사이징 비용을 줄일 수 있다.

```
ConcurrentHashMap<String, Data> map =
        new ConcurrentHashMap<>(100_000);
```

`initialCapacity`는 정확한 내부 배열 크기가 아니라 필요한 엔트리 수를 수용하기 위한 힌트이다.

### 핫 키와 핫 bin

`ConcurrentHashMap`도 동일 키에 대한 수정이 매우 많으면 해당 키를 중심으로 직렬화된다.

예를 들어 모든 요청이 다음 키 하나만 갱신한다면:

```
map.merge("global-count", 1L, Long::sum);
```

맵이 아무리 커도 경합이 한곳에 집중된다. 이런 경우 `LongAdder`처럼 경합 분산에 특화된 구조가 더 적합할 수 있다.

---

## 잘못된 예시

```
// 1. 복합 연산이 자동으로 원자적이라고 생각
if (!map.containsKey(key)) {
    map.put(key, value);
}

// 2. 읽기-수정-쓰기를 분리
map.put(key, map.get(key) + 1);

// 3. 값 객체도 자동으로 스레드 안전하다고 생각
map.computeIfAbsent(key, k -> new ArrayList<>()).add(value);

// 4. compute 안에서 느린 작업 수행
map.computeIfAbsent(key, k -> callRemoteApi(k));

// 5. size()로 최대 크기를 강제
if (map.size() < limit) {
    map.put(key, value);
}

// 6. 반복 결과를 완전한 스냅샷으로 간주

// 7. 변경 가능한 객체를 키로 사용

// 8. 최신 구현도 Segment 기반이라고 생각
```

---

## 사용 여부

적합한 경우:

- 여러 스레드가 공유하는 조회 테이블
- 사용자 ID → 세션 같은 레지스트리
- 키별 상태 관리
- 중복 생성 방지
- 빈도수 집계
- 읽기가 많고 수정도 동시에 발생하는 데이터
- 완벽한 전체 스냅샷보다 높은 동시성이 중요한 경우

다른 구조가 더 나을 수 있는 경우:

- 단일 스레드만 접근: `HashMap`
- 키 정렬이 필요: `ConcurrentSkipListMap`
- 만료 시간, 최대 크기, 자동 제거가 필요한 캐시: 별도의 캐시 구현
- 여러 키에 걸친 트랜잭션이 필요: 외부 락이나 다른 상태 관리 설계
- 정확한 전역 스냅샷이 필요: 별도의 동기화 또는 불변 스냅샷 구조
