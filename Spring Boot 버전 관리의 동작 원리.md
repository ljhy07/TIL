## 핵심 개념

Spring Boot 프로젝트에서 보통 의존성에 버전을 쓰지 않는다.

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

이게 가능한 이유는 Spring Boot가 다음을 제공하기 때문이다.

- `spring-boot-dependencies`
    - Spring, Jackson, Tomcat, Hibernate, Reactor, Micrometer 등 주요 라이브러리 버전 목록을 가진 BOM
- `spring-boot-starter-parent`
    - Maven에서 부모 POM으로 사용
    - 내부적으로 Boot의 의존성 관리와 빌드 기본값을 제공
- Gradle `org.springframework.boot` 플러그인
    - Boot 플러그인 버전에 맞는 BOM을 가져와 의존성 버전을 맞춤

## BOM이란?

BOM은 **Bill of Materials**의 약자이다.

쉽게 말하면 다음과 같은 “권장 버전 표”다.

| 라이브러리            | Boot 3가 관리하는 버전      |
| ---------------- | -------------------- |
| Spring Framework | Boot 버전에 맞는 Spring 6 |
| Jackson          | Boot가 테스트한 버전        |
| Hibernate        | Boot가 테스트한 버전        |
| Tomcat           | Boot가 테스트한 버전        |
| SLF4J            | Boot가 테스트한 버전        |
BOM은 의존성을 자동으로 추가하지 않는다.
“해당 의존성을 사용할 때 어떤 버전을 쓸지”만 정한다.

## Maven에서의 동작

### 1. `spring-boot-starter-parent` 사용

가장 일반적인 방식이다.

```
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.x.x</version>
</parent>
```

이렇게 하면 Maven은 Spring Boot 부모 POM을 통해 다음을 얻는다.

- `spring-boot-dependencies` 기반 의존성 버전 관리
- Java 버전, 인코딩 등 기본 빌드 설정
- Maven plugin 버전 관리

그래서 의존성 선언에서 버전을 생략할 수 있다.

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

Maven은 Boot BOM을 보고 관련 라이브러리 버전을 결정한다.

### 2. 부모 POM을 못 쓰는 경우

회사 표준 parent POM이 있으면 Spring Boot parent를 상속하지 못할 수 있다.  
그럴 때는 `dependencyManagement`에 BOM을 직접 import한다.

```
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>3.x.x</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

이 방식도 의존성 버전 관리는 가능하지만, `spring-boot-starter-parent`가 주는 일부 빌드 기본값은 직접 설정해야 한다.

## Gradle에서의 동작

```
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.x.x'
    id 'io.spring.dependency-management' version '1.x.x'
}
```

`io.spring.dependency-management` 플러그인을 함께 적용하면 Spring Boot Gradle 플러그인이 현재 Boot 버전에 맞는 `spring-boot-dependencies` BOM을 가져온다.

또는 Gradle의 native BOM 기능을 사용할 수도 있다.

```
dependencies {
    implementation platform(org.springframework.boot.gradle.plugin.SpringBootPlugin.BOM_COORDINATES)
}
```

Maven은 `dependencyManagement` 중심이고, Gradle은 플러그인 또는 `platform()`을 통해 BOM을 적용한다.  
결과적으로는 둘 다 “Boot 버전에 맞는 의존성 버전 세트”를 사용한다.

## Starter의 역할

`spring-boot-starter-web` 같은 starter는 버전 관리 도구가 아니라 **의존성 묶음**이다.

예를 들어 `spring-boot-starter-web`은 대략 다음 계열의 의존성을 끌고 온다.

- Spring MVC
- Embedded Tomcat
- Jackson
- Validation 관련 의존성

starter는 “무엇을 가져올지”를 정하고, BOM은 “각각 몇 버전을 쓸지”를 정한다.

## 버전 결정 우선순위

1. 개발자가 의존성을 선언한다.
2. 버전을 직접 적지 않으면 Spring Boot BOM의 관리 버전을 사용한다.
3. 직접 버전을 명시하면 그 버전이 우선될 수 있다.
4. 다만 Boot가 테스트한 조합에서 벗어나므로 호환성 문제가 생길 수 있다.

예시:

```
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
</dependency>
```

이 경우 Jackson 버전은 Boot BOM이 정한다.

반대로:

```
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>직접 지정한 버전</version>
</dependency>
```

이렇게 하면 Boot의 권장 조합을 벗어날 수 있다.

Spring Boot 공식 문서도 관리 버전 override는 가능하지만 주의하라고 안내한다.  
각 Boot 릴리스는 특정 third-party dependency 조합으로 설계되고 테스트되기 때문이다.