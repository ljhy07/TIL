한 줄 정리
> **JavaBean**은 “자바 객체를 일정한 규약에 맞게 만든 클래스”이다.

## 1. Java Bean이란?

Java Bean은 Java에서 재사용 가능한 컴포넌트를 만들기 위해 정한 객체 작성 규약이다.

### 대표적인 조건

1. **기본 생성자**

```
public User() {}
```

2. **필드는 보통 private**

```
private String name;
```

3. **getter / setter 제공** 

```
public String getName() {
	return name;
}
    
public void setName(String name) {
	this.name = name;
}
```

4. **직렬화 가능하도록 만드는 경우**

```
public class User implements Serializable
```


예시:

```
public class User {
    private Long id;
    private String name;
    private boolean active;

    public User() {
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public boolean isActive() {
        return active;
    }

    public void setActive(boolean active) {
        this.active = active;
    }
}
```

중요한 점은 Java Bean의 “프로퍼티”는 필드 자체가 아니라 **getter/setter 메서드 이름을 기준으로 인식된다**는 것이다.

```
private String username;

public String getName() {
    return username;
}
```

이 경우 JavaBeans 관점에서는 필드명이 `username`이어도 프로퍼티 이름은 `name`으로 볼 수 있다.

JavaBean은 주로 다음 용도로 많이 쓰인다.

- JSP/Servlet에서 데이터 전달 객체
- DTO, VO 형태의 단순 데이터 객체
- 프레임워크가 리플렉션으로 값을 주입하거나 읽는 대상
- IDE, GUI Builder, ORM, JSON 변환기 등이 다루기 쉬운 객체

JavaBean의 핵심은 **객체를 표준화된 방식으로 읽고 쓸 수 있게 하는 규약**이다.