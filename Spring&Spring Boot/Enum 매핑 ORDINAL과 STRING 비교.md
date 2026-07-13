## ORDINAL
`EnumType.ORDINAL`은 enum의 순서 번호를 DB에 저장한다.

```
@Enumerated(EnumType.ORDINAL)
private OrderStatus status;
```

DB에는 아래처럼 저장된다.

| Enum 값      | DB 저장값 |
| ----------- | ------ |
| `READY`     | `0`    |
| `PAID`      | `1`    |
| `SHIPPED`   | `2`    |
| `CANCELLED` | `3`    |

장점은 저장 공간이 작고 숫자라서 간단하다는 점이다. 하지만 치명적인 단점이 있다. enum 선언 순서가 바뀌거나 중간에 값이 추가되면 기존 데이터의 의미가 깨진다.

예를 들어 나중에 이렇게 바꾸면:

```
public enum OrderStatus {
    READY,
    CONFIRMED,
    PAID,
    SHIPPED,
    CANCELLED
}
```

기존 DB의 `1`은 원래 `PAID`였는데, 이제는 `CONFIRMED`가 된다. 데이터가 조용히 잘못 해석되는 아주 위험한 상황이다.

## STRING
`EnumType.STRING`은 enum 이름을 DB에 저장한다.

```
@Enumerated(EnumType.STRING)
private OrderStatus status;
```

DB에는 이렇게 저장된다.

|Enum 값|DB 저장값|
|---|---|
|`READY`|`"READY"`|
|`PAID`|`"PAID"`|
|`SHIPPED`|`"SHIPPED"`|
|`CANCELLED`|`"CANCELLED"`|

장점은 enum 순서가 바뀌거나 중간에 값이 추가되어도 기존 데이터가 안전하다는 점이다. DB를 직접 봐도 의미가 명확하다.

단점은 `ORDINAL`보다 저장 공간을 더 쓰고, enum 이름을 바꾸면 기존 DB 값과 매핑이 깨질 수 있다는 점이다. 예를 들어 `PAID`를 `PAYMENT_COMPLETED`로 rename하면 DB에 남아 있는 `"PAID"`를 읽지 못할 수 있다.

## 비교 요약

|기준|`ORDINAL`|`STRING`|
|---|---|---|
|DB 저장값|숫자|문자열|
|가독성|낮음|높음|
|저장 공간|작음|상대적으로 큼|
|enum 순서 변경|위험함|안전함|
|enum 중간 삽입|위험함|안전함|
|enum 이름 변경|영향 적음|위험함|
|실무 추천|거의 비추천|일반적으로 추천|

## 정리

대부분의 경우에는 `STRING`을 쓰는 것이 좋다.

```
@Enumerated(EnumType.STRING)
@Column(nullable = false)
private OrderStatus status;
```

특히 주문 상태, 회원 등급, 결제 상태, 게시글 상태처럼 도메인 의미가 중요한 값은 `STRING`이 훨씬 안전하다.

`ORDINAL`은 enum 값이 절대 추가/삭제/재정렬되지 않는다는 강한 보장이 있거나, 이미 레거시 DB가 숫자로 설계되어 있는 경우에만 조심해서 쓰는 편이 좋다.

더 안정적인 대안은 enum에 명시적인 코드를 두고 `AttributeConverter`로 저장하는 방식이다.

```
public enum OrderStatus {
    READY("R"),
    PAID("P"),
    SHIPPED("S"),
    CANCELLED("C");

    private final String code;

    OrderStatus(String code) {
        this.code = code;
    }

    public String getCode() {
        return code;
    }
}
```

이 방식은 enum 이름을 바꿔도 DB 값은 `"P"`처럼 유지할 수 있어서, 장기적으로는 `STRING`보다도 더 안정적인 경우가 많다.

결론적으로는:

```
// 기본 추천
@Enumerated(EnumType.STRING)
private OrderStatus status;
```

`ORDINAL`은 저장 공간 몇 바이트 아끼려다가 데이터 의미가 깨질 수 있어서, 보통은 피하는 게 맞다.