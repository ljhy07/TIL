# Java `HashMap`

`HashMap`은 **키(key)를 이용해 값(value)을 저장하고 빠르게 조회하는 자료구조**이다.

예를 들어 회원 ID로 회원 정보를 찾거나, 상품 코드로 가격을 조회하거나, 단어별 등장 횟수를 집계할 때 사용한다.

```
Map<String, Integer> prices = new HashMap<>();

prices.put("apple", 1_000);
prices.put("banana", 2_000);

System.out.println(prices.get("apple")); // 1000
```

아래에서는 기본 사용법과 내부 동작을 연결해 설명한다. 내부 구현에 관한 내용은 **OpenJDK 25 기준**이며, API가 보장하는 동작과 구현 세부 사항은 구분해야 한다.

## 기본 구조: 키와 값의 쌍

`HashMap<K, V>`에서 `K`는 키의 타입, `V`는 값의 타입이다.

```
Map<String, Integer> ages = new HashMap<>();
```

이 선언은 다음 의미이다.

|구성|의미|
|---|---|
|`Map`|키와 값으로 데이터를 관리하는 인터페이스|
|`HashMap`|해시 테이블을 사용하는 `Map` 구현체|
|`String`|키의 타입|
|`Integer`|값의 타입|

제네릭 타입에는 `int` 같은 기본형 대신 `Integer` 같은 참조형을 사용한다.

```
ages.put("민수", 25); // int 25가 Integer로 자동 박싱됨
```

보통 변수 타입은 `Map`, 실제 객체는 `HashMap`으로 선언한다. 이렇게 하면 사용하는 코드가 특정 구현체에 덜 의존하게 된다.

참고로 `Map`은 Java 컬렉션 프레임워크에 포함되지만, `List`나 `Set`과 달리 `Collection` 인터페이스를 상속하지 않는다.

## 핵심 특징

|특징|설명|
|---|---|
|키 중복 불가|같은 키에 다시 저장하면 값이 교체됨|
|값 중복 가능|서로 다른 키가 같은 값을 가질 수 있음|
|순서 보장 없음|삽입 순서나 정렬 순서를 보장하지 않음|
|`null` 허용|`null` 키 하나와 여러 `null` 값을 허용|
|빠른 키 조회|해시가 잘 분산되면 조회·삽입이 평균적으로 빠름|
|동기화되지 않음|여러 스레드가 공유하며 수정할 때 별도 대책 필요|

여기서 **순서가 없다는 말은 매번 무작위로 나온다는 뜻이 아니다.** 특정 실행에서 일정한 순서로 보여도, 그 순서에 의존하면 안 된다는 뜻이다.

### 같은 키로 다시 저장하면?

```
Map<String, Integer> scores = new HashMap<>();

scores.put("민수", 80);
Integer oldScore = scores.put("민수", 95);

System.out.println(oldScore);          // 80
System.out.println(scores.get("민수")); // 95
System.out.println(scores.size());      // 1
```

`put()`은 기존 값을 반환한다. 해당 키가 없었다면 `null`을 반환하지만, 기존 값 자체가 `null`이었던 경우에도 반환값은 `null`이다.

## 자주 사용하는 메서드

|메서드|기능|
|---|---|
|`put(key, value)`|추가 또는 기존 값 교체|
|`get(key)`|값 조회|
|`getOrDefault(key, defaultValue)`|키가 없으면 기본값 반환|
|`containsKey(key)`|키 존재 여부 확인|
|`containsValue(value)`|값 존재 여부 확인|
|`remove(key)`|항목 삭제|
|`size()`|저장된 항목 수 반환|
|`isEmpty()`|비어 있는지 확인|
|`clear()`|모든 항목 삭제|

```
Map<String, Integer> stock = new HashMap<>();

stock.put("노트북", 10);
stock.put("키보드", 30);

System.out.println(stock.get("노트북"));            // 10
System.out.println(stock.get("마우스"));            // null
System.out.println(stock.getOrDefault("마우스", 0)); // 0

System.out.println(stock.containsKey("키보드")); // true

stock.remove("키보드");

System.out.println(stock.size()); // 1
```

### `get()`의 `null`은 두 가지 의미가 있다

```
Map<String, String> users = new HashMap<>();
users.put("user1", null);

System.out.println(users.get("user1")); // null: 키는 있지만 값이 null
System.out.println(users.get("user2")); // null: 키가 없음
```

이를 구분하려면 `containsKey()`를 사용한다.

```
System.out.println(users.containsKey("user1")); // true
System.out.println(users.containsKey("user2")); // false
```

또한 `getOrDefault()`는 **키가 있는데 값이 `null`이면 `null`을 반환**한다.

```
System.out.println(users.getOrDefault("user1", "기본값")); // null
System.out.println(users.getOrDefault("user2", "기본값")); // 기본값
```

따라서 `null`이 반환될 수 있는 값을 바로 기본형으로 받으면 주의해야 한다.

```
Map<String, Integer> ages = new HashMap<>();

// null을 int로 언박싱하려 하므로 NullPointerException 발생
int age = ages.get("없는회원");
```

## 내부 동작: 키로 저장 위치를 계산한다

`HashMap`은 내부적으로 **버킷(bucket) 배열**을 사용헌다. 버킷은 항목이 들어가는 칸이라고 생각하면 된다.

키를 조회하는 흐름은 다음과 같다.

```
키
 ↓
hashCode()로 해시 코드 계산
 ↓
해시 값을 보정
 ↓
버킷 위치 계산
 ↓
해당 버킷 안에서 키 비교
 ↓
값 반환
```

예를 들어 다음 데이터를 저장했다고 가정해 본다.

```
map.put("apple", 1000);
map.put("banana", 2000);
map.put("orange", 3000);
```

내부 구조는 개념적으로 다음과 같습니다. 아래 버킷 번호는 설명용이다.

```
버킷 배열
┌───────┬─────────────────────────────────┐
│   0   │ 비어 있음                       │
│   1   │ ("apple", 1000)                  │
│   2   │ 비어 있음                       │
│   3   │ ("banana", 2000) → ("orange", 3000) │
│  ...  │ ...                             │
└───────┴─────────────────────────────────┘
```

`"apple"`을 찾을 때 모든 항목을 처음부터 확인하는 대신, 계산된 버킷으로 바로 이동한다. 이것이 빠른 조회의 핵심이다.

### 해시 보정과 버킷 계산

OpenJDK 구현은 개념적으로 다음 계산을 사용한다.

```
int h = key.hashCode();
int hash = h ^ (h >>> 16);

int index = (capacity - 1) & hash;
```

- `>>> 16`: 상위 비트를 오른쪽으로 이동한다.
- `^`: XOR 연산으로 상위 비트 정보를 하위 비트에 섞는다.
- `&`: 배열 범위에 해당하는 버킷 인덱스를 구한다.

이 계산을 위해 내부 배열 크기는 일반적으로 `16`, `32`, `64`처럼 **2의 거듭제곱**으로 관리힌다. `null` 키는 별도로 해시 값 `0`으로 처리한다.

## `hashCode()`와 `equals()`의 역할

두 메서드는 서로 다른 일을 한다.

|메서드|역할|
|---|---|
|`hashCode()`|데이터를 찾을 후보 위치를 결정하는 데 사용|
|`equals()`|키가 논리적으로 같은지 판단하는 데 사용|

**해시 코드가 같다고 같은 키는 아니다.**

```
해시 코드가 같음
    ↓
같은 키일 수도 있고, 다른 키일 수도 있음
    ↓
equals()로 동등성 확인
```

반대로 다음 규칙은 반드시 지켜야 한다.

> `equals()`가 `true`인 두 객체는 반드시 같은 `hashCode()`를 반환해야 한다.

이를 수식으로 표현하면 다음과 같다.

```
a.equals(b) == true
```

이면 반드시

```
a.hashCode() == b.hashCode()
```

여야 합니다. 반대 방향은 성립하지 않는다.

### 직접 만든 객체를 키로 사용할 때

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
        return Objects.hashCode(id);
    }
}
```

이제 서로 다른 객체라도 `id`가 같으면 같은 키로 사용할 수 있다.

```
Map<UserKey, String> users = new HashMap<>();

users.put(new UserKey(1), "민수");

System.out.println(users.get(new UserKey(1))); // 민수
```

`equals()`와 `hashCode()`를 재정의하지 않은 일반 클래스는 기본적으로 객체의 정체성을 기준으로 비교하므로, 내용이 같아 보이는 새 객체로 조회해도 찾지 못할 수 있다.

### 키의 비교 기준은 저장 후 바꾸지 않는다

키의 `equals()`나 `hashCode()`에 사용하는 필드를 저장 후 변경하면 조회가 깨질 수 있다.

```
// id가 변경 가능한 필드라고 가정
UserKey key = new UserKey(1);
map.put(key, "민수");

// 이후 id를 2로 변경한다면:
// 저장 당시 위치와 조회할 때 계산하는 위치가 달라질 수 있음
```

그래서 키는 `String`, `Integer`, 또는 위 예제처럼 비교에 사용하는 필드가 변경되지 않는 객체로 만드는 편이 안전하다.

값 객체는 변경할 수 있다. 문제의 핵심은 **키를 찾는 기준이 바뀌는 것**이다.

## 해시 충돌은 어떻게 처리할까?

**서로 다른 키가 같은 버킷에 배정되는 상황**을 해시 충돌이라고 한다.

충돌은 `hashCode()`가 같을 때뿐 아니라, 서로 다른 해시 값이 같은 버킷 인덱스로 계산될 때도 발생한다.

### 충돌이 적을 때: 연결 리스트

같은 버킷 안에서 항목들을 연결해 관리한다.

```
bucket[3]
   ↓
[keyA, valueA] → [keyB, valueB] → [keyC, valueC]
```

조회할 때는 해당 버킷의 후보들을 비교한다. 충돌이 지나치게 많으면 확인해야 할 항목도 늘어난다.

### 충돌이 많을 때: 트리

OpenJDK는 조건을 만족하면 해당 버킷을 **레드-블랙 트리**로 바꾼다.

주요 구현 상수는 다음과 같다.

|상수|값|의미|
|---|---|---|
|`TREEIFY_THRESHOLD`|8|트리 전환 판단에 사용하는 기준|
|`MIN_TREEIFY_CAPACITY`|64|트리 전환을 허용하는 최소 배열 용량|

다만 “항목이 정확히 8개가 되는 순간 항상 트리로 바뀐다”고 이해하면 부정확하다. 일반적인 `put()` 경로에서는 기존 8개 항목에 9번째를 추가할 때 전환을 시도하며, 배열 용량이 64보다 작으면 우선 배열을 확장한다.

이 조건들은 API 계약이 아닌 구현 세부 사항이다.

## 용량, 적재율, 리사이징

세 용어를 구분하면 성능을 이해하기 쉽다.

|용어|의미|
|---|---|
|`size`|실제 저장된 키·값 쌍의 개수|
|`capacity`|내부 버킷 배열의 크기|
|`load factor`|용량 확장 기준을 계산하는 적재율|

기본 설정은 다음과 같다.

```
기본 초기 용량: 16
기본 적재율: 0.75
확장 기준: 16 × 0.75 = 12
```

기본 설정에서 일반적인 `put()`으로 서로 다른 키를 계속 추가하면, 13번째 항목을 넣을 때 용량이 보통 32로 늘어난다.

**적재율 0.75는 버킷의 75%가 실제로 채워졌다는 뜻이 아니다.** 하나의 버킷에 여러 항목이 들어갈 수 있으므로, 전체 항목 수와 배열 크기를 기준으로 판단한다.

### 리사이징이란?

내부 배열을 확장하고 항목들을 새 배열의 적절한 위치에 배치하는 작업이다. 이 과정은 일반적인 삽입보다 비용이 크다.

따라서 예상 데이터 수가 많으면 초기 용량을 고려하는 것이 도움이 된다.

```
Map<String, Integer> map = new HashMap<>(1000);
```

여기서 `1000`은 **확장 없이 저장할 항목 수를 직접 지정하는 값이 아니라 초기 용량 요청값**이다.

Java 19 이상에서는 예상 항목 수로 생성할 수 있다.

```
Map<String, Integer> map = HashMap.newHashMap(1000);
```

용량을 지나치게 크게 잡으면 메모리를 더 사용하고 순회 비용도 증가하므로, 무조건 크게 설정하는 것이 좋지는 않다.

## 시간 복잡도

`n`은 저장된 항목 수, `c`는 내부 배열 용량이다.

|연산|일반적인 비용|
|---|---|
|`get(key)`|평균 `O(1)`|
|`containsKey(key)`|평균 `O(1)`|
|`put(key, value)`|평균·분할상환 관점에서 `O(1)`|
|`remove(key)`|평균 `O(1)`|
|`containsValue(value)`|전체 탐색이 필요할 수 있어 `O(c + n)`|
|전체 순회|`O(c + n)`|
|`size()`|`O(1)`|

평균 `O(1)`은 **해시가 잘 분산되고, 키의 해시 계산과 비교 비용이 일정하다는 가정**에서 이해해야 한다.

개별 `put()`은 리사이징 때문에 비쌀 수 있다. 트리 버킷 역시 모든 종류의 키에 대해 최악 `O(log n)` 조회를 무조건 보장하는 것은 아니다. 같은 해시를 가진 키들을 비교 순서로 구분하기 어려우면 더 많은 노드를 탐색할 수 있다.

## 순회하는 방법

### 키와 값을 함께 사용할 때

```
Map<String, Integer> scores = new HashMap<>();

scores.put("민수", 90);
scores.put("지수", 85);

for (Map.Entry<String, Integer> entry : scores.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}
```

키와 값이 모두 필요하면 `entrySet()`이 적합한다.

### 키만 또는 값만 사용할 때

```
for (String name : scores.keySet()) {
    System.out.println(name);
}

for (Integer score : scores.values()) {
    System.out.println(score);
}
```

### 람다로 순회하기

```
scores.forEach((name, score) ->
    System.out.println(name + ": " + score)
);
```

`keySet()`, `values()`, `entrySet()`은 독립적인 복사본이 아니라 **원본 맵과 연결된 뷰(view)**이다.

```
Set<String> names = scores.keySet();

names.remove("민수");

System.out.println(scores.containsKey("민수")); // false
```

뷰에서 항목을 삭제하면 원본 맵에서도 삭제된다.

## 순회 중 삭제할 때 주의할 점

다음처럼 순회 도중 맵에서 직접 항목을 삭제하면 `ConcurrentModificationException`이 발생할 수 있다.

```
for (String key : scores.keySet()) {
    if (scores.get(key) < 90) {
        scores.remove(key); // 잘못된 삭제 방식
    }
}
```

**이 예외는 단일 스레드에서도 발생할 수 있다.**

조건에 따라 삭제하려면 다음처럼 작성하면 된다.

```
scores.entrySet().removeIf(entry -> entry.getValue() < 90);
```

명시적인 반복자가 필요하다면 `Iterator.remove()`를 사용한다.

```
Iterator<Map.Entry<String, Integer>> iterator =
    scores.entrySet().iterator();

while (iterator.hasNext()) {
    Map.Entry<String, Integer> entry = iterator.next();

    if (entry.getValue() < 90) {
        iterator.remove();
    }
}
```

이러한 fail-fast 동작은 잘못된 수정을 발견하기 위한 장치이며, 예외 발생 자체가 항상 보장되지는 않는다. 동시성 제어 수단으로 사용하면 안 된다.

## 실무에서 유용한 활용 패턴

### 기본값 등록: `putIfAbsent()`

```
Map<String, Integer> settings = new HashMap<>();

settings.putIfAbsent("timeout", 30);
settings.putIfAbsent("timeout", 60);

System.out.println(settings.get("timeout")); // 30
```

기존 값이 `null`이면 새 값을 저장한다.

### 단어 빈도 집계: `merge()`

```
String[] words = {"java", "spring", "java", "sql", "spring", "java"};

Map<String, Integer> counts = new HashMap<>();

for (String word : words) {
    counts.merge(word, 1, Integer::sum);
}

System.out.println(counts.get("java"));   // 3
System.out.println(counts.get("spring")); // 2
System.out.println(counts.get("sql"));    // 1
```

`merge()`는 기존 값이 없거나 `null`이면 전달한 값을 저장하고, 기존 값이 있으면 함수를 적용한다. 병합 함수가 `null`을 반환하면 해당 항목을 삭제한다.

### 그룹별 데이터 모으기: `computeIfAbsent()`

```
Map<String, List<String>> teams = new HashMap<>();

teams.computeIfAbsent("개발팀", key -> new ArrayList<>()).add("민수");
teams.computeIfAbsent("개발팀", key -> new ArrayList<>()).add("지수");
teams.computeIfAbsent("디자인팀", key -> new ArrayList<>()).add("수진");

System.out.println(teams.get("개발팀")); // [민수, 지수]
```

키의 값이 없거나 `null`일 때만 함수를 실행해 값을 만든다. 함수가 `null`을 반환하면 새 값을 등록하지 않는다.