`@ElementCollection`과 별도 자식 엔티티는 둘 다 DB에서는 별도 테이블을 사용할 수 있다. 핵심 차이는 테이블 구조보다 **자식 데이터에 독립적인 식별자와 생명주기가 필요한가**이다.

## `@ElementCollection`

부모 엔티티가 소유하는 **값 타입 컬렉션**을 저장할 때 사용한다.

- 원시/기본 타입: `String`, `Integer`, enum 등
- `@Embeddable` 값 객체
- 각 요소는 독립적인 엔티티가 아님
- 별도 `@Id`가 없음
- 부모 없이 독립적으로 존재하거나 공유되지 않음
- 기본 fetch는 `LAZY`지만, JPA 명세상 `LAZY`는 provider에 대한 힌트

공식 명세에서도 element collection을 기본 타입 또는 embeddable 타입의 컬렉션으로 정의하고, collection table을 이용해 매핑하도록 설명한다.

### 예제

```
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ElementCollection(fetch = FetchType.LAZY)
    @CollectionTable(
        name = "order_options",
        joinColumns = @JoinColumn(name = "order_id")
    )
    private Set<OrderOption> options = new HashSet<>();

    public void addOption(OrderOption option) {
        options.add(option);
    }

    public void removeOption(OrderOption option) {
        options.remove(option);
    }
}
```

```
@Embeddable
public class OrderOption {

    @Column(name = "option_name", nullable = false)
    private String name;

    @Column(name = "option_value", nullable = false)
    private String value;

    protected OrderOption() {
    }

    public OrderOption(String name, String value) {
        this.name = name;
        this.value = value;
    }

    // 값 객체이므로 equals/hashCode 구현 필요
}
```

대략적인 테이블 구조는 다음과 같다.

```
orders
------
id PK

order_options
-------------
order_id FK
option_name
option_value
```

`order_options`의 행은 독립적인 객체가 아니라 `Order`에 포함된 값이다.

### 적합한 사례

- 회원의 관심 키워드
- 상품 태그
- 주소 이력 중 독립 식별이 필요 없는 값
- 주문 당시 복사된 옵션 스냅샷
- 단순 문자열이나 코드 목록

예를 들어 주문 옵션이 주문 당시의 `"색상=검정"`이라는 값일 뿐이고, 개별 옵션 행을 수정·참조·감사할 필요가 없다면 `@ElementCollection`이 자연스럽다.

---

## 별도 자식 엔티티

자식에게 `@Entity`와 `@Id`를 부여하고 `@OneToMany` / `@ManyToOne` 관계로 매핑한다.

```
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(
        mappedBy = "order",
        cascade = CascadeType.ALL,
        orphanRemoval = true
    )
    private List<OrderOptionEntity> options = new ArrayList<>();

    public void addOption(OrderOptionEntity option) {
        options.add(option);
        option.assignOrder(this);
    }

    public void removeOption(OrderOptionEntity option) {
        options.remove(option);
        option.assignOrder(null);
    }
}
```

```
@Entity
@Table(name = "order_options")
public class OrderOptionEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @Column(name = "option_name", nullable = false)
    private String name;

    @Column(name = "option_value", nullable = false)
    private String value;

    @Version
    private Long version;

    protected OrderOptionEntity() {
    }

    public OrderOptionEntity(String name, String value) {
        this.name = name;
        this.value = value;
    }

    void assignOrder(Order order) {
        this.order = order;
    }
}
```

테이블 구조는 다음처럼 된다.

```
order_options
-------------
id PK
order_id FK
option_name
option_value
version
```

자식 행마다 `id`가 있으므로 다음과 같은 작업이 가능하다.

```
orderOptionRepository.findById(optionId);
orderOptionRepository.deleteById(optionId);
```

또한 다른 엔티티가 특정 `OrderOptionEntity`를 참조하거나, 자식 단위로 변경 이력과 낙관적 락을 적용할 수 있다.

---

## `@ElementCollection` vs 자식 엔티티

|기준|`@ElementCollection`|별도 자식 엔티티|
|---|---|---|
|객체 성격|값 객체|엔티티|
|독립 `@Id`|없음|있음|
|DB 테이블|보통 별도 collection table|별도 entity table|
|독립 조회|제한적|자연스러움|
|개별 행 참조|어려움|ID로 가능|
|다른 엔티티와 관계|제한적|자유롭게 관계 설정 가능|
|생명주기|부모에 완전히 종속|독립 또는 부모 종속으로 설계 가능|
|삭제 처리|컬렉션에서 제거하면 값 행 제거|`orphanRemoval`, cascade 또는 직접 삭제 필요|
|변경 이력/감사|요소 단위 관리가 어려움|자식 단위 관리 가능|
|낙관적 락|요소 자체에 `@Version` 불가|자식마다 `@Version` 가능|
|업데이트 효율|컬렉션 형태/provider에 따라 여러 행 재처리 가능|ID 기준 단건 수정이 용이|
|모델 복잡도|단순함|관계·생명주기 관리가 필요|
|Repository|부모 Repository 중심|자식 Repository 생성 가능|

## 선택 기준

다음 질문 하나가 가장 중요하다.

> “이 자식 데이터를 ID로 개별 식별해야 하는가?”

### `@ElementCollection`을 선택

- 자식은 부모의 일부일 뿐이다.
- 자식 ID가 도메인에서 의미가 없다.
- 다른 객체가 특정 자식을 참조하지 않는다.
- 항상 부모를 통해 추가·삭제·조회한다.
- 개별 요소보다 컬렉션 전체가 작고 단순하다.

### 별도 자식 엔티티를 선택

- 개별 자식의 ID가 필요하다.
- 자식을 직접 검색하거나 수정한다.
- 데이터 양이 많아 페이징해야 한다.
- 다른 엔티티가 자식을 참조한다.
- 자식마다 상태, 생성일, 수정일, 버전, 변경 이력이 필요하다.
- 자식이 앞으로 더 복잡해질 가능성이 높다.

## 정리

`@ElementCollection`은 “간단하니까” 선택하기보다는 **정말 값 타입인가**를 기준으로 선택하는 것이 좋습니다.

특히 다음 요구가 하나라도 있다면 별도 엔티티가 안전하다.

- 자식 목록 페이징
- 자식 단건 수정 API
- 자식별 감사 로그
- 자식별 상태 전이
- 외부 시스템에서 자식 ID 사용
- 다른 테이블에서 자식 참조
- 컬렉션에 수백~수천 건 저장

반대로 태그, 키워드, 단순 설정값처럼 소량이며 부모에 완전히 종속된 데이터라면 `@ElementCollection`이 모델을 훨씬 간결하게 만든다.