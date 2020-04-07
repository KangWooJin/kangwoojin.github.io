---
title: "[Spring] Spring boot, gradle에서 query dsl 셋팅하기" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-04-06 23:00:00+09:00 
excerpt: Spring Boot, gradle에서 query dsl 셋팅히가
---

## 들어가며
예전에 [query dsl 관련 셋팅]({% post_url jpa/2020-02-23-query-dsl-setting %})을 올려두었는데
컴파일 시점에 model을 생성하고 모델이 변경되었을 때 찾지 못하는 문제가 있었다.

내가 잘 못 셋팅했을 수도 있지만 해당 방식보다 좀 더 쉽고 최근 방식이 있는거 같아서 또 다른 방법을 올려둔다.

## 셋팅하기

```groovy
buildscript {
    ext {
        springBootVersion = '2.2.6.RELEASE'
        querydslPluginVersion = '1.0.10'
    }
    repositories {
        mavenCentral()
        maven { url "https://plugins.gradle.org/m2/" } // plugin 저장소
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath "io.spring.gradle:dependency-management-plugin:1.0.7.RELEASE"
        classpath("gradle.plugin.com.ewerk.gradle.plugins:querydsl-plugin:${querydslPluginVersion}")
    }
}

apply plugin: 'java'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'
apply plugin: "com.ewerk.gradle.plugins.querydsl"

group = 'kangwoojin.github.io'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

compileQuerydsl{
    options.annotationProcessorPath = configurations.querydsl
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
    querydsl.extendsFrom compileClasspath
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("com.querydsl:querydsl-jpa") // querydsl
    implementation("com.querydsl:querydsl-apt") // querydsl

    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    compileOnly 'org.projectlombok:lombok'
    runtimeOnly 'com.h2database:h2'
    annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
    annotationProcessor 'org.projectlombok:lombok'
    testCompileOnly("org.projectlombok:lombok")
    testAnnotationProcessor("org.projectlombok:lombok")
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
}

test {
    useJUnitPlatform()
}

def querydslSrcDir = 'src/main/generated'

querydsl {
    library = "com.querydsl:querydsl-apt"
    jpa = true
    querydslSourcesDir = querydslSrcDir
}

sourceSets {
    main {
        java {
            srcDirs = ['src/main/java', querydslSrcDir]
        }
    }
}
```

gradle의 전체 설정인데 여기서 중요한 부분만 알아보자.

```groovy
buildscript {
    ext {
        springBootVersion = '2.2.6.RELEASE'
        querydslPluginVersion = '1.0.10'
    }
    repositories {
        mavenCentral()
        maven { url "https://plugins.gradle.org/m2/" } // plugin 저장소
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        classpath "io.spring.gradle:dependency-management-plugin:1.0.7.RELEASE"
        classpath("gradle.plugin.com.ewerk.gradle.plugins:querydsl-plugin:${querydslPluginVersion}")
    }
}
```

groovy에 대한 지식이 없어서 buildscript가 정확히 무엇을 하는지는 모르겠지만
여기서 필요한 것은 `repositories`와 `querydsl plugin` 이다.

query dsl의 plugin을 사용하기 위해서는 특정 저장소를 써야 다운로드 받아 지기에 반드시 plugin 저장소를 등록해야 한다.

querydsl-plugin 관련해 포스트를 쓰는 시점 최신은 1.0.10이며, 해당 옵션을 추가 해줘야 나중에 나올 옵션들을 사용할 수 있다.

```groovy
apply plugin: 'java'
apply plugin: 'org.springframework.boot'
apply plugin: 'io.spring.dependency-management'
apply plugin: "com.ewerk.gradle.plugins.querydsl"
```

여기서는 기본적으로 intellij에서 생성해주는 java, spring 설정을 제외하고 querydsl 관련
plugin을 추가해줘야 한다.

```groovy
compileQuerydsl{
    options.annotationProcessorPath = configurations.querydsl
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
    querydsl.extendsFrom compileClasspath
}
```

gradle 5.0이상부터는 해당 옵션을 넣어줘야 한다고 한다.

```groovy
dependencies {
    // web, jpa, h2, lombok 추가시 기본 설정
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    runtimeOnly 'com.h2database:h2'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
    annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }
    
    // query dsl 설정 추가
    implementation("com.querydsl:querydsl-jpa") // querydsl
    implementation("com.querydsl:querydsl-apt") // querydsl
    // test시에도 query dsl 모델을 사용하기 위해서는 해당 옵션을 추가 해줘야 한다.
    testCompileOnly("org.projectlombok:lombok")
    testAnnotationProcessor("org.projectlombok:lombok")
}
```
intellij에서 자동으로 추가해주는 옵션들을 제외하고는 query dsl을 위한 jpa, apt와
lombok 관련된 설정이다.

테스트 시에도 Qmodel을 사용하기 위해서는 lombok 설정을 넣으니 해결이 되었는데, 정확한 이유는 모르겠다.

```groovy
def querydslSrcDir = 'src/main/generated'

querydsl {
    library = "com.querydsl:querydsl-apt"
    jpa = true
    querydslSourcesDir = querydslSrcDir
}

sourceSets {
    main {
        java {
            srcDirs = ['src/main/java', querydslSrcDir]
        }
    }
}
```

query dsl plugin을 추가했을 때 사용할 수 있는 querydsl 파트인데
자세한 옵션들은 [여기](https://github.com/ewerk/gradle-plugins/tree/master/querydsl-pluginhttps://github.com/ewerk/gradle-plugins/tree/master/querydsl-plugin)에서 확인할 수 있다.

여기서 가장 중요한 옵션은 jpa를 사용한다면 jpa 부분을 true로 해줘야 하고,
 source dir를 원하는 곳으로 하고 싶으면 `querydslSourcesDir`에 원하는 값으로 설정해줘야 한다.  

## 마치며
실제로 query dsl을 사용하고 있는데 gradle의 query dsl만 generate해주는 기능이 없어
해당 방식을 공부하고 실제로 적용해보려고 한다.

gradle의 generate해주는 기능이 추가가 되면 매번 컴파일시에 query dsl model로 인해
컴파일이 느려지거나 에러가 발생하는 부분이 줄어들거라 생각한다!

- - - 
[querydsl](https://github.com/KangWooJin/spring-study/tree/master/querydsl)
관련 example code는 github에 올려두었으니 참고~!
 