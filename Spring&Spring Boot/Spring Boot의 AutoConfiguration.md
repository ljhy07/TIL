## 핵심
Spring Boot Auto-configuration은 “classpath, Bean 존재 여부, 설정값, 실행 환경을 보고 Spring이 필요한 설정을 자동으로 등록해 주는 메커니즘”이다.  
`spring.factories`는 예전 방식의 자동 설정 등록 파일이고, 최신 Spring Boot에서는 auto-configuration 등록에 `AutoConfiguration.imports`를 쓴다.

## Auto-configuration

`@SpringBootApplication`은 내부적으로 다음 세 가지를 포함한다.

```
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

여기서 핵심은 `@EnableAutoConfiguration`이다.

`@EnableAutoConfiguration`이 켜지면 Spring Boot는 “현재 프로젝트에 어떤 라이브러리가 들어와 있는지”를 보고 자동으로 설정 클래스를 import한다.

예를 들어:
- `spring-boot-starter-web`이 있으면 `DispatcherServlet`, Tomcat, Spring MVC 관련 Bean 자동 구성
- `spring-boot-starter-data-jpa`가 있으면 `EntityManagerFactory`, `TransactionManager` 등 자동 구성
- `H2`, `PostgreSQL`, `MySQL` 드라이버가 있으면 `DataSource` 자동 구성 시도
- `spring-boot-starter-security`가 있으면 기본 Security filter chain 자동 구성

즉, 개발자가 직접 이런 설정을 일일이 작성하지 않아도 된다.

```
@Configuration
@EnableWebMvc
@ComponentScan
public class WebConfig {
    // DispatcherServlet, ViewResolver, HandlerMapping...
}
```

Boot에서는 이런 설정 대부분을 자동으로 해 줍니다.

## 동작 흐름
### 흐름

```
@SpringBootApplication
        ↓
@EnableAutoConfiguration
        ↓
AutoConfigurationImportSelector
        ↓
META-INF/spring/...AutoConfiguration.imports 읽기
        ↓
자동 설정 후보 클래스 목록 수집
        ↓
@ConditionalOnClass, @ConditionalOnMissingBean 등 조건 검사
        ↓
조건에 맞는 설정만 ApplicationContext에 등록
```

중요한 점은 **모든 자동 설정이 무조건 적용되는 것이 아니라 조건부로 적용된다**는 것이다.

예:
```
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
public class DataSourceAutoConfiguration {
}
```

이 경우 `DataSource` 관련 클래스가 classpath에 있을 때만 자동 설정 후보가 된다.

## 조건 애노테이션

Auto-configuration의 핵심은 `@Conditional...` 계열이다.

자주 쓰는 것들:

```
@ConditionalOnClass
```

특정 클래스가 classpath에 있을 때 활성화된다.

```
@ConditionalOnMissingBean
```

사용자가 직접 등록한 Bean이 없을 때만 기본 Bean을 등록한다.  
이게 Spring Boot의 “기본값은 제공하지만, 사용자가 원하면 쉽게 덮어쓴다”는 철학의 핵심이다.

```
@ConditionalOnProperty
```

설정값에 따라 활성화된다.

```
@ConditionalOnBean
```

특정 Bean이 이미 존재할 때 활성화된다.

```
@ConditionalOnWebApplication
```

웹 애플리케이션일 때만 활성화된다.

예시:
```
@AutoConfiguration
@ConditionalOnClass(MyClient.class)
@EnableConfigurationProperties(MyClientProperties.class)
public class MyClientAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyClient myClient(MyClientProperties properties) {
        return new MyClient(properties.getUrl(), properties.getTimeout());
    }
}
```

이 설정은 다음 조건에서만 동작된다.

- `MyClient` 클래스가 classpath에 있음
- 사용자가 `MyClient` Bean을 직접 만들지 않았음
- `MyClientProperties`를 통해 설정값 바인딩 가능

## spring.factories

`spring.factories`는 `META-INF/spring.factories` 위치에 두는 SPI 등록 파일이다.

예전 Spring Boot 1.x, 2.x에서는 auto-configuration 클래스를 여기에 등록했다.

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.autoconfig.MyAutoConfiguration,\
com.example.autoconfig.MyWebAutoConfiguration
```

이 방식은 Spring Boot가 `SpringFactoriesLoader`를 통해 `EnableAutoConfiguration` 키에 등록된 클래스들을 읽고 자동 설정 후보로 삼는 구조이다.

하지만 최신 방식에서는 auto-configuration 등록 용도로 `spring.factories`를 쓰지 않는다.

## 최신 Auto-configuration 등록 방식

Spring Boot 2.7부터 새 방식이 도입되었고, Spring Boot 3.x 이후에는 auto-configuration 등록은 보통 이 파일을 사용한다.

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

파일 내용은 단순히 클래스명을 한 줄에 하나씩 적는다.

```
com.example.autoconfig.MyAutoConfiguration
com.example.autoconfig.MyWebAutoConfiguration
```

예전 방식:
```
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.autoconfig.MyAutoConfiguration
```

최신 방식:
```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.autoconfig.MyAutoConfiguration
```

### 정리

|Spring Boot 버전|Auto-configuration 등록 방식|
|---|---|
|Boot 1.x, 2.x|`META-INF/spring.factories`|
|Boot 2.7|`AutoConfiguration.imports` 도입, 마이그레이션 구간|
|Boot 3.x+|`AutoConfiguration.imports` 권장/표준|

단, `spring.factories` 자체가 완전히 사라진 것은 아니다.
예를 들어 `EnvironmentPostProcessor`, `FailureAnalyzer` 같은 일부 Spring Boot SPI 등록에는 여전히 사용된다.
```
org.springframework.boot.env.EnvironmentPostProcessor=\
com.example.MyEnvironmentPostProcessor
```

## @AutoConfiguration

`@AutoConfiguration`은 자동 설정 클래스를 표시하기 위한 애노테이션이다.

예전에는 보통 이렇게 썼다.
```
@Configuration(proxyBeanMethods = false)
public class MyAutoConfiguration {
}
```

최신 Spring Boot에서는 이렇게 쓴다.
```
@AutoConfiguration
public class MyAutoConfiguration {
}
```

`@AutoConfiguration`은 자동 설정 클래스임을 더 명확하게 표현하고, 순서 제어도 지원한다.

```
@AutoConfiguration(after = DataSourceAutoConfiguration.class)
public class MyJpaAutoConfiguration {
}
```
또는:
```
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
@AutoConfiguration
public class MyJpaAutoConfiguration {
}
```

순서 제어 애노테이션:
```
@AutoConfigureBefore
@AutoConfigureAfter
@AutoConfigureOrder
```

주의할 점은 자동 설정 순서는 **Bean 생성 순서 자체라기보다 Bean definition 등록 순서**에 영향을 준다.

## Starter와 Auto-configuration의 관계

`starter`는 보통 의존성 묶음이다.

예:
```
implementation "org.springframework.boot:spring-boot-starter-web"
```

이 starter는 내부적으로 다음 같은 것들을 가져온다.

- Spring MVC
- Jackson
- Validation
- Embedded Tomcat
- 관련 auto-configuration

실제 자동 설정 클래스는 보통 `spring-boot-autoconfigure` 또는 별도 autoconfigure 모듈에 들어 있다.

즉:
```
starter = dependency 모음
autoconfigure = 조건부 설정 코드
```

라이브러리를 직접 만든다면 보통 구조를 이렇게 나눈다.

```
my-client
my-client-autoconfigure
my-client-spring-boot-starter
```

## Auto-configuration 디버깅
자동 설정이 왜 적용됐는지, 왜 안 됐는지 보고 싶으면 다음처럼 실행할 수 있다.

```
java -jar app.jar --debug
```

그러면 condition evaluation report가 출력된다.

Actuator를 쓰면 `/actuator/conditions`에서도 확인할 수 있다.

```
/actuator/conditions
```

여기서 어떤 auto-configuration이 positive match인지, negative match인지 확인할 수 있다.