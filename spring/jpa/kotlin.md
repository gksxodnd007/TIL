### Kotlin으로 JPA를 이용해 개발할 때 주의할 점

#### 인자없는 기본생성자
- JPA의 Entity들은 기본적으로 인자없는 기본생성자가 필요하다. 하지만 코틀린에서 주생성자가 있다면 기본생성자가 없다. 디폴트 파라미터를 통해 기본생성자가 생성되게 할 수 있지만 프로퍼티가 많아지면 모든 Entity에 디폴트 파라미터를 넣기 힘들어진다. 다음과 같이 해결 할 수 있다.

```groovy
dependencies {
  ...
  // JPA entity들은 기본적으로 기본생성자가 필요하다. 하지만 주 생성자가 존재하면 기본생성자가 없다.
  // @Entity가 붙은 클래스들에 한해서 자동으로 인자없는 생성자를 추가해준다.
  classpath("org.jetbrains.kotlin:kotlin-noarg:${kotlinVersion}")
}

// @Entity가 붙은 클래스들에 한해서 자동으로 인자없는 생성자를 추가해준다.
apply plugin: 'kotlin-jpa'
```

- build.gradle에 위와 같이 설정해두면 코틀린 코드가 자바바이트코드로 컴파일될때 기본 생성자가 생긴다.

#### 지연로딩 문제
- Hibernate는 지연로딩을 위해 Entity들을 상속하여 프록시를 만든다. 하지만 코틀린에서는 클래스의 기본 상속 제어자가 final이기 때문에 지연로딩으로 설정해도 프록시를 만들지 못해 지연로딩이 되지 않는 문제가 발생한다. 이런 문제는 다음과 같이 해결 할 수 있다.

```groovy
dependencies {
  ...
  // kotlin에서 클래스는 기본 final이기 때문에 JPA에서 지연로딩시 entity를 상속받아 처리하는 proxy를 이용할 수 없다.
  // 즉, 지연로딩을 할 수 없다. 이를 해결해주는 컴파일러 플러그인이다. 모든 entity를 open시켜준다.
  classpath("org.jetbrains.kotlin:kotlin-allopen:${kotlinVersion}")
}

allOpen {
    annotation "javax.persistence.Entity"
}
```

- build.gradle에 위와 같이 설정해두면 @Entity 애너테이션이 붙은 클래스들은 모두 open으로 변경된다.
- 다음은 전체 build.gradle 파일이다.

```groovy
buildscript {
    ext {
        kotlinVersion = '1.2.71'
        springBootVersion = '2.1.2.RELEASE'
    }
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:${kotlinVersion}")
        // kotlin에서 클래스는 기본 final이기 때문에 JPA에서 지연로딩시 entity를 상속받아 처리하는 proxy를 이용할 수 없다.
        // 즉, 지연로딩을 할 수 없다. 이를 해결해주는 컴파일러 플러그인이다. 모든 entity를 open시켜준다.
        classpath("org.jetbrains.kotlin:kotlin-allopen:${kotlinVersion}")
        // JPA entity들은 기본적으로 기본생성자가 필요하다. 하지만 주 생성자가 존재하면 기본생성자가 없다.
        // @Entity가 붙은 클래스들에 한해서 자동으로 인자없는 생성자를 추가해준다.
        classpath("org.jetbrains.kotlin:kotlin-noarg:${kotlinVersion}")
    }
}

apply plugin: 'kotlin'
apply plugin: 'kotlin-spring'
// @Entity가 붙은 클래스들에 한해서 자동으로 인자없는 생성자를 추가해준다.
apply plugin: 'kotlin-jpa'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'
// @Entity가 붙은 클래스들을 모두 open으로 바꿔준다
// allOpen task 확인
apply plugin: "kotlin-allopen"

group = 'coding.squid'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.8'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'com.fasterxml.jackson.module:jackson-module-kotlin'
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk8"
    implementation "org.jetbrains.kotlin:kotlin-reflect"
    runtimeOnly 'com.h2database:h2'
    runtimeOnly 'mysql:mysql-connector-java'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'io.projectreactor:reactor-test'
}

compileKotlin {
    kotlinOptions {
        freeCompilerArgs = ['-Xjsr305=strict']
        jvmTarget = '1.8'
    }
}

compileTestKotlin {
    kotlinOptions {
        freeCompilerArgs = ['-Xjsr305=strict']
        jvmTarget = '1.8'
    }
}

allOpen {
    annotation "javax.persistence.Entity"
}
```

- 샘플 코드는 https://github.com/gksxodnd007/spring-kotlin 여기서 볼 수 있다.
