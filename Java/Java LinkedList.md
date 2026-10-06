# Java의 LinkedList

`LinkedList`는 **각 원소를 노드에 저장하고, 노드들을 앞뒤로 연결하는 이중 연결 리스트**이다.

핵심 특징은 **양 끝의 삽입·삭제가 빠르지만, 인덱스로 특정 위치를 찾는 작업은 느리다**는 점이다. 또한 `List`와 `Deque`를 모두 구현하므로 리스트, 큐, 스택으로 사용할 수 있다.

---

## LinkedList란?

Java의 `LinkedList`는 `java.util` 패키지에 있다.

```
import java.util.LinkedList;

LinkedList<String> names = new LinkedList<>();
```

`<String>`은 저장할 원소의 타입이다. 정수를 저장하려면 `LinkedList<Integer>`처럼 사용한다.

주요 특징은 다음과 같다.

|특징|설명|
|---|---|
|순서 유지|리스트에 배치된 순서대로 원소를 순회|
|중복 허용|같은 값을 여러 번 저장 가능|
|`null` 허용|`null`도 원소로 저장 가능|
|크기 변경|원소 추가·삭제에 따라 크기가 바뀜|
|양방향 연결|각 노드가 이전 노드와 다음 노드를 참조|
|기본 동기화 없음|여러 스레드가 함께 수정하려면 별도 조치 필요|

여기서 **순서 유지가 자동 정렬을 의미하지는 않는다.** `30`, `10`, `20`을 차례로 추가하면 그 순서대로 저장된다.

```
LinkedList<Integer> numbers = new LinkedList<>();

numbers.add(30);
numbers.add(10);
numbers.add(20);
numbers.add(10);

System.out.println(numbers); // [30, 10, 20, 10]
```

## 내부 구조: 이중 연결 리스트

### 노드가 앞뒤의 노드를 연결한다

`LinkedList`의 구조를 단순화하면 다음과 같다.

```
first                                last
  ↓                                    ↓
 [A]  ⇄  [B]  ⇄  [C]  ⇄  [D]
```

각 노드는 세 가지 정보를 가진다.

```
┌───────────────────┐
│ prev: 이전 노드   │
│ item: 저장된 원소 │
│ next: 다음 노드   │
└───────────────────┘
```

리스트는 첫 노드인 `first`, 마지막 노드인 `last`, 원소 개수인 `size`를 관리한다. 노드 안에는 실제 원소 객체를 가리키는 참조가 들어간다.

### 원소를 추가할 때

`B`와 `C` 사이에 `X`를 넣는다고 생각해 본다.

```
추가 전: A ⇄ B ⇄ C ⇄ D
추가 후: A ⇄ B ⇄ X ⇄ C ⇄ D
```

해당 위치에 도달한 뒤에는 주변 노드의 연결을 바꾸면 된다. 뒤쪽 원소들을 하나씩 이동시킬 필요가 없다.

### 원소를 삭제할 때

`B`를 삭제하면 `A`와 `C`를 직접 연결한다.

```
삭제 전: A ⇄ B ⇄ C ⇄ D
삭제 후: A ⇄ C ⇄ D
```

**삽입·삭제 위치를 찾는 비용과, 연결을 변경하는 비용을 구분해야 한다.** 이 차이가 `LinkedList`의 성능을 이해하는 핵심이다.

## List, Queue, Deque로 사용하는 방법

`LinkedList`는 `List`와 `Deque`를 구현하며, `Deque`는 `Queue`를 상속합니다. 따라서 사용 목적에 맞는 타입으로 선언할 수 있다.

```
import java.util.Deque;
import java.util.LinkedList;
import java.util.List;
import java.util.Queue;

// 인덱스를 사용하는 리스트
List<String> list = new LinkedList<>();

// 먼저 들어온 원소를 먼저 꺼내는 큐
Queue<String> queue = new LinkedList<>();

// 양 끝에서 넣고 꺼내는 덱
Deque<String> deque = new LinkedList<>();
```

선언 타입에 따라 호출할 수 있는 메서드가 달라진다.

```
list.get(0);        // List의 인덱스 접근
queue.offer("A");  // Queue의 원소 추가
deque.addFirst("A"); // Deque의 앞쪽 추가
```

`Queue`나 `Deque` 타입으로 선언하면 `get(index)`는 호출할 수 없다. 구현 객체가 같아도 **변수의 타입이 제공하는 API를 사용하기 때문**이다.

## 기본 메서드와 사용 예제

### 리스트 메서드

|메서드|동작|반환값|
|---|---|---|
|`add(e)`|맨 뒤에 추가|`boolean`|
|`add(index, e)`|지정한 위치에 삽입|없음|
|`get(index)`|지정한 위치의 원소 조회|조회한 원소|
|`set(index, e)`|지정한 위치의 원소 교체|기존 원소|
|`remove(index)`|지정한 위치의 원소 삭제|삭제한 원소|
|`remove(Object o)`|처음 발견한 일치 원소 삭제|삭제 여부|
|`contains(o)`|원소 포함 여부 확인|`boolean`|
|`indexOf(o)`|처음 일치하는 원소의 인덱스 조회|인덱스, 없으면 `-1`|
|`size()`|원소 개수 조회|`int`|
|`isEmpty()`|비어 있는지 확인|`boolean`|
|`clear()`|모든 원소 삭제|없음|

인덱스는 `0`부터 시작한다. `get`, `set`, `remove(index)`에는 `0`부터 `size() - 1`까지 사용할 수 있고, `add(index, e)`에는 맨 뒤 삽입을 위해 `size()`도 사용할 수 있다.

```
import java.util.LinkedList;

public class Main {
    public static void main(String[] args) {
        LinkedList<String> list = new LinkedList<>();

        list.add("A");
        list.add("C");
        list.add(1, "B");

        System.out.println(list);        // [A, B, C]
        System.out.println(list.get(1)); // B

        String previous = list.set(1, "X");
        System.out.println(previous);   // B
        System.out.println(list);       // [A, X, C]

        String removed = list.remove(1);
        System.out.println(removed);    // X
        System.out.println(list);       // [A, C]
    }
}
```

### 양 끝을 다루는 메서드

|작업|앞쪽|뒤쪽|
|---|---|---|
|추가|`addFirst(e)`|`addLast(e)`|
|조회|`getFirst()`|`getLast()`|
|조회, 비어 있으면 `null`|`peekFirst()`|`peekLast()`|
|삭제|`removeFirst()`|`removeLast()`|
|삭제, 비어 있으면 `null`|`pollFirst()`|`pollLast()`|

`getFirst`, `getLast`, `removeFirst`, `removeLast`는 비어 있으면 `NoSuchElementException`을 던진다. 조회 메서드는 원소를 남겨 두고, 삭제 메서드는 원소를 꺼내면서 제거한다.

## 시간 복잡도

`n`을 원소 개수라고 할 때, 일반적인 시간 복잡도는 다음과 같다.

|연산|시간 복잡도|
|---|---|
|`add(e)`, `addFirst(e)`, `addLast(e)`|O(1)|
|`getFirst()`, `getLast()`|O(1)|
|`removeFirst()`, `removeLast()`|O(1)|
|`peek`, `poll` 계열|O(1)|
|`size()`, `isEmpty()`|O(1)|
|`get(index)`, `set(index, e)`|최악 O(n)|
|`add(index, e)`, `remove(index)`|최악 O(n)|
|`contains(o)`, `indexOf(o)`, `remove(Object o)`|최악 O(n)|
|반복자로 전체 순회|O(n)|
|`clear()`|O(n)|

이 표는 OpenJDK의 노드 탐색과 연결 변경 방식에서 도출한 것으로, 메모리 할당과 원소의 `equals()` 비용 등은 별도로 고려하지 않은 설명이다.

### 인덱스 접근이 느린 이유

```
list.get(500);
```

`LinkedList`는 인덱스만으로 해당 노드에 바로 접근할 수 없다. 연결을 따라 이동해야 한다.

다만 항상 처음부터 탐색하지는 않는다.

- 앞쪽에 가까운 인덱스이면 `first`부터 이동한다.
- 뒤쪽에 가까운 인덱스이면 `last`부터 거꾸로 이동한다.

따라서 양 끝의 접근은 빠르지만, 가운데에 가까울수록 탐색 비용이 커진다.

### “중간 삽입·삭제는 O(1)”의 정확한 의미

```
list.add(500, "X");
```

이 작업은 다음 두 단계로 이루어진다.

1. 인덱스 `500`에 해당하는 노드를 찾는다.
2. 주변 연결을 변경해 새 노드를 삽입한다.

두 번째 단계는 O(1)이지만, 첫 번째 단계 때문에 **전체 작업은 최악 O(n)**이다.

반면 반복자가 이미 해당 위치에 도달해 있다면, 그 위치에서의 삽입·삭제는 O(1)로 수행할 수 있다.

## 순회할 때는 get(i)를 반복하지 않기

### 인덱스 반복문의 문제

```
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}
```

각 `get(i)`가 해당 위치를 다시 탐색한다. 따라서 전체 순회의 탐색 비용이 **O(n²)**이 된다.

### 향상된 for문 사용

```
for (String value : list) {
    System.out.println(value);
}
```

이 방식은 반복자를 통해 다음 노드로 이동하므로 전체 순회가 **O(n)**이다.

`LinkedList`처럼 인덱스 접근 비용이 있는 리스트에서는 반복자를 사용하는 순회가 적합하다.

### 순회하면서 삭제하기

향상된 for문 안에서 `list.remove(...)`를 직접 호출하면 `ConcurrentModificationException`이 발생할 수 있다.

삭제가 필요하면 반복자의 `remove()`를 사용한다.

```
import java.util.Iterator;

// list에 저장된 문자열 중 "B"를 삭제
Iterator<String> iterator = list.iterator();

while (iterator.hasNext()) {
    String value = iterator.next();

    if ("B".equals(value)) {
        iterator.remove();
    }
}
```

`iterator.remove()`는 **직전 `next()`가 반환한 원소**를 삭제한다. `next()`를 호출하기 전에 사용하거나, 같은 원소에 대해 두 번 연속 사용하면 `IllegalStateException`이 발생한다.

## ListIterator로 순회 중 삽입·수정하기

`ListIterator`는 일반 `Iterator`보다 더 많은 기능을 제공한다.

|메서드|기능|
|---|---|
|`next()`|다음 원소로 이동|
|`previous()`|이전 원소로 이동|
|`add(e)`|커서 위치에 원소 삽입|
|`remove()`|마지막으로 반환한 원소 삭제|
|`set(e)`|마지막으로 반환한 원소 교체|

반복자의 커서는 원소 사이에 있다고 생각하면 이해하기 쉽다.

```
import java.util.Arrays;
import java.util.LinkedList;
import java.util.ListIterator;

LinkedList<String> list =
        new LinkedList<>(Arrays.asList("A", "C"));

ListIterator<String> iterator = list.listIterator();

iterator.next();   // A 반환, 커서는 A와 C 사이
iterator.add("B");  // 커서 위치에 B 삽입

System.out.println(list); // [A, B, C]

String next = iterator.next();
System.out.println(next); // C

iterator.set("D");
System.out.println(list); // [A, B, D]
```

이미 위치를 잡은 `LinkedList`의 반복자는 다음·이전 이동과 삽입·삭제를 O(1)로 수행한다. 다만 `listIterator(index)`로 중간 위치의 반복자를 처음 만들 때는 그 위치를 찾는 비용이 든다.

## 큐와 스택으로 사용하기

### 큐: FIFO

큐는 **먼저 넣은 원소를 먼저 꺼내는 구조**이다.

```
import java.util.LinkedList;
import java.util.Queue;

Queue<String> queue = new LinkedList<>();

queue.offer("A");
queue.offer("B");
queue.offer("C");

System.out.println(queue.peek()); // A, 삭제하지 않음
System.out.println(queue.poll()); // A, 삭제
System.out.println(queue.poll()); // B, 삭제
System.out.println(queue);        // [C]
```

큐 메서드는 다음처럼 구분한다.

| 작업  | 예외 방식       | 특수값 반환 방식  |
| --- | ----------- | ---------- |
| 추가  | `add(e)`    | `offer(e)` |
| 조회  | `element()` | `peek()`   |
| 삭제  | `remove()`  | `poll()`   |

빈 큐에서 `element()`와 `remove()`는 예외를 던지고, `peek()`와 `poll()`은 `null`을 반환한다. `LinkedList`에는 설정된 용량 제한이 없으므로 일반적인 추가 성공 시 `add()`와 `offer()`는 모두 `true`를 반환한다.

### 스택: LIFO

스택은 **마지막에 넣은 원소를 먼저 꺼내는 구조**이다.

```
import java.util.Deque;
import java.util.LinkedList;

Deque<String> stack = new LinkedList<>();

stack.push("A");
stack.push("B");
stack.push("C");

System.out.println(stack.peek()); // C
System.out.println(stack.pop());  // C
System.out.println(stack.pop());  // B
System.out.println(stack.pop());  // A
```

`push()`는 앞쪽에 추가하고, `pop()`은 앞쪽 원소를 제거한다. 빈 상태에서 `pop()`을 호출하면 `NoSuchElementException`이 발생한다.

## 자주 실수하는 부분

### remove(int)와 remove(Object)의 차이

정수를 저장할 때 특히 주의해야 한다.

```
LinkedList<Integer> numbers =
        new LinkedList<>(Arrays.asList(10, 20, 30));

numbers.remove(1);

System.out.println(numbers); // [10, 30]
```

`remove(1)`은 **값 `1`이 아닌 인덱스 `1`의 원소**를 삭제한다.

값을 삭제하려면 객체를 전달한다.

```
LinkedList<Integer> numbers =
        new LinkedList<>(Arrays.asList(10, 20, 30));

boolean removed = numbers.remove(Integer.valueOf(20));

System.out.println(removed); // true
System.out.println(numbers); // [10, 30]
```

같은 값이 여러 개 있으면 처음 일치하는 원소 하나만 삭제한다.

### 큐에 null을 저장하면 상태 구분이 어려워진다

`LinkedList`는 `null`을 허용하지만, 큐로 사용할 때는 피하는 편이 좋다.

```
Queue<String> queue = new LinkedList<>();

queue.offer(null);

System.out.println(queue.peek());    // null
System.out.println(queue.isEmpty()); // false
```

`peek()`의 반환값만 보면 비어 있는지, 맨 앞 원소가 `null`인지 구분할 수 없다. `poll()`에서도 같은 모호함이 생긴다.
### ConcurrentModificationException은 단일 스레드에서도 발생한다

이 예외는 여러 스레드가 있을 때만 발생하는 것이 아니다. 하나의 스레드에서도 반복 중 리스트를 직접 추가·삭제하면 발생할 수 있다.

`LinkedList`의 반복자는 이런 변경을 감지하는 **fail-fast** 방식이다. 다만 감지가 항상 보장되지는 않으며, 스레드 안전성을 제공하는 기능도 아니다.

## ArrayList와 비교하기

| 비교 항목        | ArrayList    | LinkedList        |
| ------------ | ------------ | ----------------- |
| 내부 구조        | 크기 조절 가능한 배열 | 이중 연결 노드          |
| 인덱스 조회·교체    | O(1)         | 최악 O(n)           |
| 맨 뒤 추가       | 분할 상환 O(1)   | O(1)              |
| 맨 앞 삽입·삭제    | O(n)         | O(1)              |
| 맨 뒤 삭제       | O(1)         | O(1)              |
| 중간 인덱스 삽입·삭제 | O(n)         | 최악 O(n)           |
| 메모리 구성       | 배열에 원소 참조 저장 | 원소마다 노드와 연결 참조 필요 |

**분할 상환 O(1)** 은 개별 추가에서 배열 확장으로 O(n)이 걸릴 수 있지만, 많은 추가 연산을 묶어 보면 연산당 평균 비용이 O(1)이라는 뜻이다.

구조상 `LinkedList`는 원소마다 노드 객체가 필요하므로 추가 메모리와 객체 할당 비용이 생긴다. 또한 노드들이 메모리에 흩어질 수 있어, 배열의 참조들을 연속으로 읽는 방식보다 CPU 캐시 활용이 불리할 수 있다.

따라서 **삽입·삭제가 많다는 이유만으로 LinkedList가 항상 빠르다고 판단하면 안 된다.** 수정 위치를 찾는 방식과 실제 작업 패턴을 함께 봐야 한다.

## 여러 스레드에서 사용하기

여러 스레드가 같은 리스트를 공유하고 수정한다면 동기화 방법을 정해야 한다.

한 가지 방법은 `Collections.synchronizedList()`로 감싸는 것이다.

```
import java.util.Collections;
import java.util.LinkedList;
import java.util.List;

List<String> shared =
        Collections.synchronizedList(new LinkedList<>());

shared.add("A");
shared.add("B");

synchronized (shared) {
    for (String value : shared) {
        System.out.println(value);
    }
}
```

이 래퍼는 개별 메서드 호출을 동기화하지만, **전체 순회에는 반환된 리스트를 대상으로 별도 동기화가 필요**하다. `isEmpty()` 확인 후 삭제하는 것처럼 여러 호출을 하나의 작업으로 처리할 때도 전체 작업을 보호해야 한다. 

## 어떤 경우에 선택하면 좋을까?

### 일반적인 리스트라면 ArrayList부터 검토

인덱스 조회, 데이터 저장, 전체 순회가 중심이면 `ArrayList`가 합리적인 기본 선택이다. 공식 문서도 `ArrayList`의 상수 비용이 `LinkedList`보다 낮다고 설명한다.

### 큐·스택·덱이라면 ArrayDeque부터 검토

양 끝에서 넣고 꺼내는 용도라면 `ArrayDeque`를 먼저 검토할 수 있다.

```
Queue<String> queue = new ArrayDeque<>();
Deque<String> stack = new ArrayDeque<>();
```

`ArrayDeque`는 `null`을 허용하지 않으며, 공식 문서는 큐로 사용할 때 `LinkedList`보다 빠를 가능성이 높다고 설명한다.

### LinkedList가 잘 맞는 경우

**리스트를 순차적으로 탐색하면서, 이미 위치를 잡은 `ListIterator`로 삽입·삭제를 반복하는 경우**에는 연결 리스트의 장점이 드러난다.

선택할 때는 다음 두 가지를 먼저 확인하면 된다.

- 원소를 찾을 때 인덱스로 접근하는가, 반복자로 이동하는가?
- 삽입·삭제가 양 끝이나 반복자의 현재 위치에서 일어나는가?