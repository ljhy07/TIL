# Spring Boot 버전 관리의 핵심 원리

> Spring Boot 버전 하나를 선택하면, 그 버전과 함께 검증된 Spring Framework 및 수많은 라이브러리 버전의 조합이 BOM을 통해 결정된다.

대략 다음 순서로 동작한다.

```
Spring Boot 버전
    ↓
spring-boot-dependencies BOM 선택
    ↓
Spring Framework / Jackson / Hibernate / Tomcat 등의 권장 버전 제공
    ↓
Starter가 실제 의존성들을 추가
    ↓
Maven 또는 Gradle이 전체 의존성 그래프를 해석
    ↓
최종 compileClasspath / runtimeClasspath 결정
```

Spring Boot는 각 릴리스마다 Spring 모듈과 서드파티 라이브러리의 검증된 버전 목록을 제공하며, 이를 `spring-boot-dependencies` BOM으로 배포한다. Boot 버전을 올리면 이 목록도 함께 바뀐다.

---

## 종류의 버전을 구분

|종류|예시|의미|
|---|---|---|
|애플리케이션 버전|`1.2.0`|우리 서비스의 배포 버전|
|Spring Boot 버전|`4.1.0`|사용할 Boot 세대와 관리되는 의존성 조합|
|라이브러리 버전|Jackson, Hibernate 등|대부분 Boot BOM이 관리|
|빌드 도구·플러그인 버전|Maven, Gradle, Boot plugin|빌드 자체의 동작과 호환성 결정|

예를 들어 Maven의 다음 두 버전은 완전히 다른 의미이다.

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.1.0</version> <!-- Spring Boot 버전 -->
</parent>

<groupId>com.example</groupId>
<artifactId>order-service</artifactId>
<version>1.7.2</version> <!-- 우리 애플리케이션 버전 -->
```

Gradle도 마찬가지다.

```
plugins {
    id("org.springframework.boot") version "4.1.0"
}

version = "1.7.2"
```

---

## BOM, Parent, Starter, Plugin은 서로 다른 개념

이 네 가지를 혼동하면 Spring Boot 의존성 관리가 이해되지 않는다.

### BOM

BOM은 Bill of Materials의 약자로, 의존성의 버전표다.

```
spring-core       → 특정 버전
spring-security   → 특정 버전
jackson-databind  → 특정 버전
hibernate-core    → 특정 버전
tomcat            → 특정 버전
```

중요한 점은 다음과 같다.

> BOM은 라이브러리를 추가하지 않는다. 라이브러리가 필요할 때 사용할 버전만 제공한다.

따라서 BOM에 Jackson이 등록되어 있어도 Jackson 의존성이 자동으로 들어오는 것은 아니다.

### Starter

Starter는 실제 의존성 묶음을 추가하는 POM이다.

예를 들어:

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

이 Starter가 Spring Data JPA, Hibernate, JDBC, 트랜잭션 관련 모듈 등을 전이 의존성으로 가져온다. 각 모듈의 버전은 BOM이 정한다.

즉:

```
Starter = 무엇을 가져올지 결정
BOM     = 어떤 버전을 사용할지 결정
```

Spring 공식 Starter는 일관되게 관리된 전이 의존성을 제공한다.

### Maven Parent

`spring-boot-starter-parent`는 BOM보다 더 많은 설정을 제공한다.

- `spring-boot-dependencies`의 의존성 관리
- Java 컴파일 기본 설정
- UTF-8 인코딩
- `-parameters`
- 리소스 필터링
- Maven 플러그인 버전과 기본 설정
- `spring-boot-maven-plugin`의 `repackage` 실행 설정

따라서 Parent를 쓰는 것과 BOM만 import하는 것은 동일하지 않다.

### Spring Boot 빌드 플러그인

Maven의 `spring-boot-maven-plugin`이나 Gradle의 `org.springframework.boot` 플러그인은 다음 작업을 담당한다.

- 실행 가능한 fat JAR 생성
- 애플리케이션 실행
- `bootJar`, `bootRun`, `repackage`
- OCI 이미지 생성
- AOT 관련 작업

특히 Maven에서는 다음 플러그인만 추가한다고 의존성 관리가 활성화되는 것이 아니다.

```
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```

Maven 의존성 관리는 Parent 또는 BOM이 담당한다.

---

# Maven에서의 동작 원리

## Starter Parent 상속

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.1.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webmvc</artifactId>
    </dependency>

    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

Starter와 PostgreSQL 버전을 작성하지 않은 이유는 Boot Parent가 상속한 BOM에 버전이 정의되어 있기 때문이다.

## Parent가 있다면 BOM만 import

Maven 프로젝트는 Parent를 하나만 가질 수 있다. 사내 공통 Parent를 사용해야 한다면 Boot BOM만 import한다.

```
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>4.1.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

이 경우 얻는 것:

- Spring Boot의 의존성 버전 관리

얻지 못하는 것:

- Boot Parent의 plugin management
- 컴파일러와 리소스 기본 설정
- `repackage` 실행 설정
- 기타 Maven 빌드 기본값

따라서 플러그인을 명시적으로 설정해야 한다.

```
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <version>4.1.0</version>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

## `dependencies`와 `dependencyManagement`의 차이

```
<dependencyManagement>
    ...
</dependencyManagement>
```

여기에 등록된 의존성은 실제 classpath에 추가되지 않는다. 버전과 범위의 기본값만 제공한다.

```
<dependencies>
    ...
</dependencies>
```

여기에 선언하거나 다른 라이브러리가 전이 의존성으로 가져와야 실제 classpath에 들어온다.

## Maven의 최종 버전 선택 규칙

BOM이나 `dependencyManagement`가 없다면 Maven은 기본적으로 dependency tree에서 프로젝트와 가장 가까운 버전을 선택한다. 같은 깊이라면 먼저 선언된 것이 선택된다.

```
application
├── library-a
│   └── common-lib:2.0
└── library-b
    └── common-lib:1.0
```

경로 깊이가 같으면 선언 순서가 영향을 줄 수 있다. 반면 `dependencyManagement`에 `common-lib:2.1`을 선언하면 전이 의존성에서 발견되는 버전을 2.1로 통제할 수 있다.

우선순위를 단순화하면 다음과 같다.

```
현재 POM의 명시적인 버전 또는 dependencyManagement
    > Parent에서 상속받은 dependencyManagement
    > 전이 의존성의 버전 중 Maven mediation 결과
```

추가로 알아야 할 점:

- 여러 Maven BOM이 같은 라이브러리를 관리하면 import 순서가 영향을 준다.
- Maven에서는 같은 위치에서 import된 BOM끼리 충돌할 경우 먼저 import된 BOM의 항목이 사용될 수 있다.
- 프로젝트의 `dependencyManagement`에 직접 선언하면 import된 BOM보다 명시적으로 우선시킬 수 있다.
- 프로젝트의 `dependencyManagement`는 Maven 빌드 플러그인의 내부 의존성에는 적용되지 않는다.

---

# Gradle에서의 동작 원리

Gradle에는 크게 두 가지 방식이 있다.

## Spring Dependency Management Plugin 방식

```
plugins {
    java
    id("org.springframework.boot") version "4.1.0"
    id("io.spring.dependency-management") version "1.1.7"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webmvc")
    runtimeOnly("org.postgresql:postgresql")
}
```

Spring Boot 플러그인과 `io.spring.dependency-management` 플러그인을 함께 적용하면, Boot 플러그인 버전에 대응하는 `spring-boot-dependencies` BOM이 자동으로 import된다.

장점:

- Maven과 유사한 동작
- 버전 없이 의존성 선언 가능
- BOM 속성을 통한 버전 재정의 가능

```
extra["slf4j.version"] = "원하는-버전"
```

여러 BOM을 import할 때 Spring Dependency Management Plugin은 순서대로 처리하며, 같은 의존성을 관리하면 마지막 BOM의 관리 정보가 사용된다. Maven BOM import와 세부 우선순위가 다르다는 점에 주의해야 한다. [Spring Dependency Management Plugin 문서](https://docs.spring.io/dependency-management-plugin/docs/current/reference/html/)

## Gradle 네이티브 `platform` 방식

```
plugins {
    java
    id("org.springframework.boot") version "4.1.0"
}

dependencies {
    implementation(
        platform(
            org.springframework.boot.gradle.plugin.SpringBootPlugin.BOM_COORDINATES
        )
    )

    implementation("org.springframework.boot:spring-boot-starter-webmvc")
}
```

또는 명시적으로:

```
dependencies {
    implementation(platform("org.springframework.boot:spring-boot-dependencies:4.1.0"))
}
```

Spring 공식 문서는 네이티브 BOM 방식이 더 빠른 빌드를 제공할 가능성이 있다고 설명한다. 하지만 동작 의미가 다르다.

### `platform`과 `enforcedPlatform`

```
implementation(platform("group:bom:version"))
```

- BOM 버전은 기본적으로 constraint 또는 권장값이다.
- 다른 강한 제약이나 의존성이 더 높은 버전을 요구하면 다른 버전이 선택될 수 있다.

```
implementation(enforcedPlatform("group:bom:version"))
```

- BOM 버전을 강제한다.
- 다른 버전 요청을 덮어쓴다.
- 라이브러리 프로젝트에서 사용하면 강제 조건이 소비자에게 전파될 수 있으므로 신중해야 한다.

이 차이는 [Spring Boot Gradle 의존성 관리 문서](https://docs.spring.io/spring-boot/gradle-plugin/managing-dependencies.html)에 명시되어 있다.

### Version Catalog는 BOM이 아니다

`libs.versions.toml`은 의존성 좌표를 중앙에서 관리하고 타입 안전한 별칭을 제공한다.

```
[libraries]
jackson = { module = "com.fasterxml.jackson.core:jackson-databind" }
```

하지만 Version Catalog 자체는 dependency graph의 버전 정렬을 보장하지 않는다. BOM이나 Gradle platform은 실제 해석 과정에 constraint로 참여한다. [Gradle Catalog와 Platform 차이](https://docs.gradle.org/current/userguide/centralizing_catalog_platform.html)

---

# 버전을 생략할 수 있는 경우와 없는 경우

## Boot가 관리하는 의존성

```
implementation("org.springframework.boot:spring-boot-starter-data-jpa")
implementation("org.postgresql:postgresql")
testImplementation("org.testcontainers:junit-jupiter")
```

해당 Boot BOM에 등록되어 있다면 버전을 생략한다.

## Boot가 관리하지 않는 의존성

```
implementation("com.example:some-library:1.4.2")
```

BOM에 없다면 버전을 직접 지정하거나 그 라이브러리의 공식 BOM을 import해야 한다.

현재 Boot가 관리하는 항목은 공식 [Managed Dependency Coordinates](https://docs.spring.io/spring-boot/appendix/dependency-versions/coordinates.html)에서 확인할 수 있다.

### 원칙

- 버전을 생략했는데 빌드가 성공한다고 무조건 Boot가 관리한 것은 아니다.
- 사내 platform이나 다른 BOM이 버전을 제공했을 수도 있다.
- 실제 출처와 최종 버전은 dependency report로 확인해야 한다.

---

# 의존성 버전을 임의로 덮어쓰면 위험한 이유

다음과 같이 개별 라이브러리를 최신 버전으로 올리는 것은 문법적으로 가능하다.

```
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>...</version>
</dependency>
```

하지만 Boot는 단순히 “대략 최신인 버전들”을 모은 것이 아니라 특정 조합을 테스트한다.

예를 들면 다음 사이에는 런타임 계약이 존재한다.

```
Spring Framework ↔ Spring Security
Spring Data JPA  ↔ Hibernate
Jackson core     ↔ Jackson modules
Tomcat           ↔ Servlet API
Micrometer       ↔ Actuator
JUnit Jupiter    ↔ JUnit Platform
```

개별 버전 재정의로 나타날 수 있는 오류:

- `NoSuchMethodError`
- `NoClassDefFoundError`
- `AbstractMethodError`
- Bean 생성 실패
- 직렬화 결과 변화
- Hibernate 쿼리 및 스키마 동작 변화
- 테스트에서는 성공하지만 운영 runtimeClasspath에서 실패

Spring Boot는 특히 Spring Framework 버전을 직접 지정하지 말 것을 강하게 권고한다.

## 재정의가 필요한 경우의 우선순위

1. 먼저 같은 Boot 라인의 최신 patch 버전으로 올린다.
2. 필요한 수정 버전이 Boot BOM에 반영됐는지 확인한다.
3. Spring Cloud 등 연계 프로젝트의 호환성을 확인한다.
4. 그래도 필요하면 최소 범위로 재정의한다.
5. Jackson, Reactor처럼 여러 모듈이 같이 움직이는 라이브러리는 개별 JAR보다 해당 BOM 또는 Boot의 버전 속성을 사용한다.
6. 이유와 제거 조건을 빌드 파일에 기록한다.
7. 실제 runtimeClasspath와 통합 테스트를 검증한다.

예:

```
<properties>
    <!-- CVE 대응. Boot 4.x.y 반영 후 제거 -->
    <jackson-bom.version>...</jackson-bom.version>
</properties>
```

현재 제공되는 override 속성은 [Version Properties 목록](https://docs.spring.io/spring-boot/appendix/dependency-versions/properties.html)에서 확인할 수 있다.

---

# Spring Cloud 버전은 Boot 버전과 별도로 맞춰야 한다

Spring Cloud는 자체 release train과 BOM을 사용한다.

```
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

Spring Cloud 버전은 “최신 버전”을 고르는 것이 아니라 현재 Boot 버전과 호환되는 release train을 골라야 한다.

2026년 8월 공식 매핑의 주요 예시는 다음과 같다.

|Spring Boot|Spring Cloud|
|---|---|
|Boot 4.0.x / 4.1.x|Cloud 2025.1.x, Boot 4.1은 2025.1.2부터|
|Boot 3.5.x|Cloud 2025.0.x|

정확한 조합은 매번 Spring Cloud 공식 호환표를 확인해야 한다.

Spring Cloud 호환성 검사기가 실행을 막는 경우, 검사기를 끄는 것은 일반적으로 해결책이 아니다.

```
spring.cloud.compatibility-verifier.enabled=false
```

이 설정은 불일치를 숨길 뿐 라이브러리 호환성을 만들어주지 않는다.

Spring AI, Spring Modulith, AWS·Azure 관련 Starter 등도 별도의 호환표나 BOM을 가질 수 있다.

---

# Java와 빌드 도구 버전도 호환성 행렬의 일부다

Spring Boot 버전만 맞아도 되는 것이 아니다.

```
Spring Boot
├── Java 버전
├── Spring Framework 버전
├── Maven 또는 Gradle 버전
├── Servlet 스펙
├── Embedded server
├── Kotlin 버전
├── Spring Cloud release train
└── GraalVM / Native Build Tools
```

2026년 8월 기준 Spring Boot 4.1.0 공식 요구사항은 다음과 같다.

- Java 17 이상, Java 26까지 호환
- Spring Framework 7.0.8 이상
- Maven 3.6.3 이상
- Gradle 8.14 이상 또는 9.x
- Servlet 6.1 기반

여기서도 세 버전을 구분해야 한다.

- 빌드를 실행하는 JDK
- 소스와 바이트코드의 target Java 버전
- 운영에서 실행하는 JRE/JDK

BOM은 Java 런타임을 관리하지 않는다. CI, 개발 환경, Docker base image, 운영 환경의 Java 버전을 별도로 일치시켜야 한다.

---

# Major, Minor, Patch의 의미

Spring Boot 버전 형식은 다음과 같다.

```
<major>.<minor>.<patch>-<qualifier>
```

예:

```
4.1.0
4.1.1
4.2.0-M1
4.2.0-RC1
4.2.0-SNAPSHOT
```

Spring Boot는 엄격한 Semantic Versioning이라고 보기는 어렵지만 버전 숫자로 업그레이드 부담을 표현한다.

- Patch: 버그 수정 중심이며 가능한 한 drop-in 호환을 지향
- Minor: 새 기능과 의존성 minor/major 변화가 있을 수 있음
- Major: 큰 호환성 변경을 예상해야 함
- `M`: milestone
- `RC`: release candidate
- `SNAPSHOT`: 내용이 계속 바뀔 수 있는 개발 버전

Spring Boot의 공식 정책상 patch 릴리스에서는 서드파티 라이브러리를 주로 patch 수준으로 올리고, 더 큰 버전 변화는 Boot minor 또는 major 릴리스에서 수행한다. [Spring Boot 버전 정책](https://github.com/spring-projects/spring-boot/wiki/Supported-Versions)

운영에서는 다음을 피하는 것이 좋다.

```
4.+
LATEST
[4.0,5.0)
4.2.0-SNAPSHOT
```

동적 버전과 Snapshot은 같은 소스에서 다른 빌드 결과를 만들 수 있기 때문이다.

---

# 안전한 업그레이드 순서

## Patch 업그레이드

예:

```
4.1.0 → 4.1.1
```

일반적인 순서:

1. Boot 버전만 변경한다.
2. dependency tree 변경을 확인한다.
3. 전체 테스트를 실행한다.
4. 애플리케이션 컨텍스트를 실제로 시작한다.
5. DB migration, 직렬화, 보안 필터, 메시징, 관측 기능을 확인한다.
6. Docker와 운영 JDK에서도 실행한다.

## Minor 업그레이드

예:

```
4.0.x → 4.1.x
```

추가로 해야 할 일:

- release notes 확인
- deprecated API 제거 여부 확인
- 설정 프로퍼티 변경 확인
- Spring Cloud release train 확인
- 빌드 도구 최소 버전 확인
- 관리되는 의존성 diff 확인

## Major 업그레이드

예:

```
3.5.x → 4.0.x
```

한 번에 건너뛰기보다는 다음처럼 접근하는 것이 안전하다.

```
현재 major의 최신 patch
    → deprecated API 제거
    → 다음 major
    → migration guide 적용
```

Spring Boot 4 마이그레이션 문서도 먼저 최신 3.5.x로 올린 뒤 4.0으로 이동하도록 안내한다.

설정 프로퍼티 변경은 임시로 다음 모듈을 사용해 찾을 수 있다.

```
runtimeOnly("org.springframework.boot:spring-boot-properties-migrator")
```

마이그레이션이 끝나면 반드시 제거해야 한다. [Spring Boot 업그레이드 문서](https://docs.spring.io/spring-boot/upgrading.html)

---

# 실제 선택된 버전을 확인하는 방법

빌드 파일에 적힌 버전보다 최종 해석 결과가 중요하다.

## Maven

전체 트리:

```
./mvnw dependency:tree
```

특정 의존성:

```
./mvnw dependency:tree \
  -Dincludes=com.fasterxml.jackson.core:jackson-databind
```

최종 상속·BOM 적용 결과:

```
./mvnw help:effective-pom
```

사용하지 않거나 직접 선언하지 않은 의존성 분석:

```
./mvnw dependency:analyze
```

## Gradle

전체 runtimeClasspath:

```
./gradlew dependencies --configuration runtimeClasspath
```

특정 라이브러리가 왜 그 버전으로 선택됐는지 확인:

```
./gradlew dependencyInsight \
  --dependency jackson-databind \
  --configuration runtimeClasspath
```

컴파일 classpath도 별도로 확인한다.

```
./gradlew dependencies --configuration compileClasspath
```

중요한 점은 `compileClasspath`와 `runtimeClasspath`가 다를 수 있다는 것이다. 컴파일 성공 후 운영에서 `NoSuchMethodError`가 발생한다면 `runtimeClasspath`부터 봐야 한다.