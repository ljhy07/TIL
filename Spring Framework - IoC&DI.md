# 제어의 역전 IoC
개발자가 제어해야할 요소들을 Spring Framework에서 대신 제어해준다라는 의미이다.

## IoC가 해주는 것들
- `Spring IoC Container`는 Spring Framework의 핵심 요소로, **객체를 생성**하고 **의존성을 구성하고 결합**하며 **생명 주기를 관리**한다.
- `Spring IoC Container`는 **DI(Dependency Injection)를 사용**해 어플리케이션에서 구성하는 컴포넌트들을 관리한다.
- `Spring IoC Container`는 xml파일과 java 코드, 어노테이션, java POJO 클래스를 통해 객체에 대한 정보를 가져오는데, 이러한 객체들을 **Bean**이라 부른다.
- **객체들의 생명주기가 개발자가 아닌 프레임워크의 제어를 통해 진행**되기 때문에, Inversion Of Control이라는 명칭을 가지게 되었다.

### POJO란?
**Plain Old Java Object**라는 의미로 말 그대로 번역하면 오래되고 간단한 자바 객체라는 의미한다. 간단하게 말하면 **객체지향적으로 구현한 자바 객체**이다.

### Bean이란?
Spring IoC Container에서 관리하고 있는 인스턴스화 된 객체를 의미한다. 애플리케이션을 실행할 때, 인스턴스화 되는 객체들은 개발자가 아닌 Spring IoC Container에서 관리하는데, 이를 Bean이라고 한다.

## IoC 동작 방식
Spring Framework 컨테이너에 비지니스 클래스(POJO)와 구성 설정(xml)이 들어가면 Spring Framework에서 해당 정보들을 바탕으로 객체를 생성하고, 해당 객체들을 Bean으로 등록 관리한다.

# 의존성 주입 DI

### 의존성이란?
객체를 생성 및 사용함에 있어 의존 관계가 있는 경우를 말한다.

## DI란?
외부에서 객체 간의 관계(의존성)를 결정해주는데, 즉, 객체를 직접 생성하는 것이 아니라 외부에서 생성 후 주잆시켜 주는 방식이다.
그래서 DI를 통해 객체 간의 관계를 동적으로 주입하여 유연성을 확보하고, 결합도를 낮출 수 있다.

## DI 방식
- Construct Injection(생성자 주입)
- Field Injection(필드 주입)
- Setter Injection(Setter 주입)

### Construct Injection(생성자 주입)
- Spring 공식 문서에서 권장하는 DI 방식이다.
- 오직 생성자 주입만이 final 키워드를 사용할 수 있다. 
- final 키워드를 사용하면 객체의 불변성(Immutability)가 보장된다.
- final 키워드 때문에 초기에 할당이 되어서 NPE(Null Point Exception)이 발생하지 않는다.

```
public class Injection {
	private InjectionService injectionService
	
	// 생성자 DI
	// @Autowired -> Spring 4.3부터는 @Authwired 생략 가능
	public Injection(InjectionService injectionService){
		this.injectionService = injectionsService;
	}
}
```

Lombok의 @RequiredArgsConstructor 사용 시
```
@RequiredArgsConstructor
public class Injection {
	private InjectionService injectionService
}
```

@RequiredArgsConstructor 사용 시 주의 사항으로는 final 키워드가 붙은 주입에만 생성자를 만들어준다. final 키워드를 사용하지 않으면 @AllArgsConstructor를 사용해야한다.

### Field Injection(필드 주입)
- Bean으로 등록하고자 하는 클래스를 Field로 선언한 뒤 @Authwired를 붙여주면 된다.
- final 키워드를 사용할 수 없다.

```
public class Injection {

	@Authwired
	private InjectionService injectionService
	
}
```

#### 장점
- 코드가 간결해진다.
#### 단점
- Solid 원칙 중에서 단일 책임 원칙(SRP)을 위반한다.
- Unit Test가 어렵다.
- final 키워드를 사용할 수 없어서 불변성이 보장되지 않는다.

### Setter Injection(Setter 주입)
- Spring에서 @Autowired 어노테이션을 사용ㅎ서 Setter 메서드를 통해 주입하는 방법이다.
- NPE가 발생할 수 있다.
- final 키워드를 사용할 수 없다.

```
public class Injection {
	private InjectionService injectionService;
	
	@Authired
	public void setInjectionService(InjectionService injectionService) {
		this.injectionService = injectionService;
	}
}
```