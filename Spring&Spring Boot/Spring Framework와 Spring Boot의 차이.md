# 1. Dependency
Spring Framework의 경우 dependency를 설정해줄 때 설정 파일이 매우 길고, 모든 dependency에 대해 버전 관리도 하나하나 해줘야한다.

아래의 예시는 Spring Framework에서 web에 대한 dependency를 추가하는 코드이다.
```
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-web</artifactId>
    <version>5.3.5</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>5.3.5</version>
</dependency>
```

Spring Boot Framework의 경우 dependency를 Spring Framework보다 쉽게 설정해 줄 수 있습니다. 버전 관리도 자동으로 해준다.

아래의 예시는 Spring Boot Framework에서 web에 대한 dependency를 추가하는 코드이다.
> implementation 'org.springframework.boot:spring-boot-starter-web'

빌드 툴을 Gradle을 사용하는 경우 위와 같이 build.gradle 파일에 dependency를 추가해주면 Spring Boot로 웹 개발을 할 때 필요한 모든 dependency를 자동으로 추가하고 관리해준다.

또 다른 예시로는 Spring Framework의 경우 test 프레임워크를 사용하려고 하는 경우 Spring Test, JUnit, Hamcrest, Mockito 등 모든 라이브러리를 추가해줘야 하지만, Spring Boot에서는 spring-boot-starter-test만 추가해주면 된다.

# 2. Configuration
Spring Framework의 경우 configuration설정을 할 때도 매우 길고, 모든 어노테이션 및 빈 등록 등을 설정해 줘야 한다.

Spring Boot Framework는 application.properties파일이나 application.yml파일에 설정하면 된다.

예를 들어 Spring Framework에서 Thymeleaf 템플릿을 사용하려면 다음과 같은 코드를 작성해야 한다.
```
@Configuration
@EnableWebMvc
public class MvcWebConfig implements WebMvcConfigurer {

    @Autowired
    private ApplicationContext applicationContext;

    @Bean
    public SpringResourceTemplateResolver templateResolver() {
        SpringResourceTemplateResolver templateResolver = 
          new SpringResourceTemplateResolver();
        templateResolver.setApplicationContext(applicationContext);
        templateResolver.setPrefix("/WEB-INF/views/");
        templateResolver.setSuffix(".html");
        return templateResolver;
    }

    @Bean
    public SpringTemplateEngine templateEngine() {
        SpringTemplateEngine templateEngine = new SpringTemplateEngine();
        templateEngine.setTemplateResolver(templateResolver());
        templateEngine.setEnableSpringELCompiler(true);
        return templateEngine;
    }

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        ThymeleafViewResolver resolver = new ThymeleafViewResolver();
        resolver.setTemplateEngine(templateEngine());
        registry.viewResolver(resolver);
    }
}
```

Spring Boot Framework에서 Thymeleaft을 사용하려면 다음과 같은 코드를 작성해야 한다.
> implementation 'org.springframework.boot:spring-boot-starter-thymeleaf

### **Spring Boot의 AutoConfiguration**
Spring Frame와 달리 Spring Boot에는 AutoConfiguration이라는 것이 있다.

Application이 있는 클래스에 @SpringBootApplication이라는 어노테이션을 확인할 수 있다.

이 어노테이션을 제거하고 프로그램을 실행하면, 일반적인 자바 프로그램과 동일하게 실행된다.

해당 어노테이션 덕분에 많은 외부 라이브러리, 내장 톰캣 서버 등이 실행될 수 있다.

> _어노테이션: 주석, 추후 특정 어노테이션을 처리하는 컴파일러가 어노테이션을 읽으면 알맞은 처리를 진행._  
> _기능이 포함된 건 아니고, 프로그램 곳곳에 분산된 기능을 한 곳에 모아서 처리하고 싶을 때 사용하기도 함._

@SpringBootApplication을 자세히 들여다보면 다음과 같은 코드를 확인할 수 있다.
![[Pasted image 20260629202452.png]]

#### @ComponentScan
@Component, @Controller, @Repository, @Service라는 어노테이션이 붙어있는 객체들을 스캔해 자동으로 Bean에 등록해준다.

#### @EnableAutoConfiguration
@ComponentScan 이후 사전에 정의한 라이브러리들을 Bean에 등록해준다.

# 3. 배포
Spring Framework로 개발한 애플리케이션의 경우, war파일을 Web Application Server에 담아 배포해야한다.

Spring Boot Framework의 경우, Tomcat 이나 Jetty 같은 내장 WAS를 가지고 있기 때문에 jar 파일로 간편하게 배포할 수 있다.

Spring Framework로 WAS를 정하고, 모든 설정을 마쳐 배포를 하는 것보다 훨씬 간단한 배포 방법이다.