# List

`List`는 “순서가 있고, 중복을 허용하는 컬렉션”이다.

```
List<String> list = new ArrayList<>();
list.add("A");
list.add("B");
list.add("A"); // 중복 허용
System.out.println(list.get(0)); // 인덱스로 접근
```

### 핵심

| 특징     | 의미                                          |
| ------ | ------------------------------------------- |
| 순서 유지  | 넣은 순서가 유지된다.                                |
| 인덱스 사용 | `get(0)`, `add(1, value)`처럼 위치 기반 접근이 가능하다. |
| 중복 허용  | 같은 객체나 같은 값을 여러 번 저장할 수 있다.                 |

`List`는 “어떻게 저장할지”를 정하지 않는다.  
그건 구현체가 정한다. 즉, `List`는 약속이고, `ArrayList`, `LinkedList` 같은 클래스가 실제 저장 방식을 담당한다.

---

# List 계층 구조

현재 JDK 기준으로 보면 큰 구조는 이렇게 볼 수 있다.

```
Iterable
  └─ Collection
       └─ SequencedCollection
            └─ List

AbstractCollection
  └─ AbstractList
       ├─ ArrayList
       ├─ Vector
       │    └─ Stack
       └─ AbstractSequentialList
            └─ LinkedList
```

`List` 공식 문서의 “All Known Implementing Classes”에는 다음이 나온다:  
`AbstractList`, `AbstractSequentialList`, `ArrayList`, `AttributeList`, `CopyOnWriteArrayList`, `LinkedList`, `RoleList`, `RoleUnresolvedList`, `Stack`, `Vector`

---

# 추상화 계층

| 타입                       | 설명                                                                                                       |
| ------------------------ | -------------------------------------------------------------------------------------------------------- |
| `List`                   | 순서, 중복, 인덱스 접근이라는 규칙을 정의하는 인터페이스이다. <br>실제 저장 방식은 없다.                                                    |
| `AbstractCollection`     | 컬렉션 공통 기능 일부를 미리 구현한 추상 클래스이다. <br>`List` 전용은 아니고 `Collection` 계열의 뼈대다.                                  |
| `AbstractList`           | 배열처럼 “인덱스로 빠르게 접근하는 구조”를 만들기 쉽게 해주는 `List`용 추상 클래스이다. <br>`get(int)`, `size()`만 구현해도 읽기 전용 리스트를 만들 수 있다. |
| `AbstractSequentialList` | `LinkedList`처럼 순차 접근이 중심인 리스트를 만들기 위한 추상 클래스이다.<br>인덱스 접근을 내부적으로 `ListIterator` 기반으로 처리한다.               |

### 요약

```
AbstractList            → 배열 기반 리스트용 뼈대
AbstractSequentialList  → 연결 리스트 기반 리스트용 뼈대
```

`AbstractList`는 `ArrayList`, `Vector` 같은 쪽에 어울리고,  
`AbstractSequentialList`는 `LinkedList`처럼 앞에서부터 따라가야 하는 구조에 어울린다.

---

# 주요 구현체

| 클래스                    | 내부 구조          | 장점                      | 단점                  | 사용 감각                 |
| ---------------------- | -------------- | ----------------------- | ------------------- | --------------------- |
| `ArrayList`            | 동적 배열          | 조회 빠름, 메모리 효율 좋음        | 중간 삽입/삭제 느림         | 대부분의 기본 선택            |
| `LinkedList`           | 이중 연결 리스트      | 앞/뒤 삽입 삭제 유리            | 인덱스 조회 느림, 메모리 부담 큼 | 큐/덱 성격이 필요할 때         |
| `Vector`               | 동기화된 동적 배열     | 메서드 동기화                 | 오래된 클래스, 성능 부담      | 레거시 코드에서 주로 만남        |
| `Stack`                | `Vector` 상속    | LIFO 구조 제공              | 오래된 설계              | 새 코드에서는 보통 `Deque` 권장 |
| `CopyOnWriteArrayList` | 변경 시 배열 복사     | 읽기 작업에 매우 안전            | 쓰기 비용 큼             | 읽기 많고 수정 적은 멀티스레드 상황  |
| `AttributeList`        | `ArrayList` 기반 | JMX 속성 목록용              | 일반 앱용 아님            | 특수 목적                 |
| `RoleList`             | `ArrayList` 기반 | JMX relation role 목록용   | 일반 앱용 아님            | 특수 목적                 |
| `RoleUnresolvedList`   | `ArrayList` 기반 | JMX unresolved role 목록용 | 일반 앱용 아님            | 특수 목적                 |

---

# ArrayList

`ArrayList`는 내부적으로 배열을 사용한다. 배열의 크기는 고정이지만, `ArrayList`는 꽉 차면 더 큰 배열을 새로 만들고 기존 요소를 복사한다.

```
List<Integer> list = new ArrayList<>();
list.add(10);
list.add(20);
list.get(1); // 빠름
```

장점은 `get(index)`가 빠르다는 것이다. 인덱스만 알면 바로 접근할 수 있다.

단점은 중간 삽입/삭제다.

```
list.add(0, 99);
```

맨 앞에 넣으면 기존 요소들을 한 칸씩 뒤로 밀어야 한다. 그래서 데이터 조회가 많고, 끝에 추가하는 일이 많은 경우 `ArrayList`가 좋다.

공식 문서도 `ArrayList`를 크기 조절 가능한 배열 구현체라고 설명하고, `get`, `set` 등은 상수 시간, 그 외 많은 작업은 선형 시간이 걸린다고 설명한다.

---

# LinkedList

`LinkedList`는 요소들이 노드로 연결된 구조다.

```
[A] <-> [B] <-> [C]
```

각 노드는 이전 노드와 다음 노드를 알고 있다. 그래서 앞이나 뒤에 추가/삭제하는 작업은 자연스럽다.

```
LinkedList<String> list = new LinkedList<>();
list.addFirst("A");
list.addLast("B");
```

하지만 `get(1000)`처럼 인덱스로 접근하면 배열처럼 바로 가지 못한다. 앞이나 뒤에서부터 노드를 따라가야 한다.

그래서 `LinkedList`는 “조회가 빠른 리스트”가 아니라 “연결 구조가 필요한 리스트”다. 또한 `List`뿐 아니라 `Deque`, `Queue` 역할도 할 수 있다.

---

# Vector와 Stack

`Vector`는 `ArrayList`와 비슷한 동적 배열이지만, 오래된 클래스이고 주요 메서드가 동기화되어 있다.

```
Vector<String> v = new Vector<>();
```

동기화는 멀티스레드에서 안전해 보이지만, 항상 필요한 것은 아니고 성능 비용도 있다. 그래서 새 코드에서는 보통 `ArrayList`를 쓰고, 필요할 때 `Collections.synchronizedList()`나 동시성 컬렉션을 선택한다.

`Stack`은 `Vector`를 상속해서 만든 LIFO 구조다.

```
Stack<Integer> stack = new Stack<>();
stack.push(1);
stack.push(2);
stack.pop(); // 2
```

하지만 공식 문서도 새 코드에서는 더 완전한 LIFO 연산을 제공하는 `Deque`, 예를 들면 `ArrayDeque` 사용을 권장한다.

---

# CopyOnWriteArrayList

`CopyOnWriteArrayList`는 멀티스레드 환경에서 읽기가 압도적으로 많고 쓰기가 적을 때 쓰는 리스트다.

이름 그대로 수정할 때마다 내부 배열을 복사한다.

```
List<String> list = new CopyOnWriteArrayList<>();
```

읽는 쪽은 안정적이다. 반복 중에 다른 스레드가 수정해도 기존 반복자는 생성 시점의 스냅샷을 본다. 대신 `add`, `set`, `remove` 같은 수정은 비싸다.

---

# 사용 기준

각 List 구현체는 아래와 같이 사용한다.

```
일반적인 리스트          → ArrayList
앞뒤 삽입/삭제, Queue/Deque → LinkedList 또는 ArrayDeque
스레드 안전 + 읽기 많음     → CopyOnWriteArrayList
오래된 코드 유지보수        → Vector, Stack 이해 필요
새 코드에서 Stack 필요      → Stack보다 ArrayDeque
```

`List` : “순서 있는 데이터 묶음”
`ArrayList` : “배열처럼 빠르게 꺼내는 리스트”
`LinkedList` : “노드가 서로 연결된 리스트”
`Vector`&`Stack` : “역사적으로 남아 있는 오래된 리스트 계열”
`CopyOnWriteArrayList` : “읽기 많은 동시성 상황을 위한 특수 리스트”