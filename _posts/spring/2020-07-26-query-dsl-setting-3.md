---
title: "[Spring] gradle에서 querydsl을 셋팅 하는 방법" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-07-26 08:00:00+09:00 
excerpt: 현재까지 gradle에서 querydsl을 셋팅하는 방법이 많이 썼지만, 가장 좋았던 방법을 소개한다.
---

## 들어가며
예전에 [query dsl 관련 셋팅]({% post_url spring/2020-04-06-query-dsl-setting %})을 올려두었다.
이전보다 더 좋은 설정 방법이 나와서 gradle에서 querydsl 관련 포스팅만 총 4번째 작성하고 있다... 이번이 마지막이 되기를!

대표적인 방법이 2가지인데, querydsl plugin을 이용하는 방법과 annotationProcessor을 이용해서
생성하는 방법이다.

이번에 작성하게 된 계기는, querydsl plugin을 이용하게 되면, querydsl로 생성된 QModel이 cache가 되지 않아
build시에 매번 생성이 되고 있는 것을 확인하고, 어떻게 하면 cache를 할 수 있을지 찾다가 발견하여 기록해 둔다.

## 어떻게 셋팅하나?

```groovy
ext {
    set('queryDSL', '4.3.1')
}

dependencies {
    // QueryDSL
    implementation "com.querydsl:querydsl-jpa:${queryDSL}"
    annotationProcessor "com.querydsl:querydsl-apt:${queryDSL}:jpa"
    annotationProcessor 'org.hibernate.javax.persistence:hibernate-jpa-2.1-api:1.0.2.Final'
    annotationProcessor 'javax.annotation:javax.annotation-api:1.3.2'
    ...
}
```

- gradle dependencies에 4개의 라이브러리만 추가하면 query dsl 셋팅이 끝난다.

## 뭐가 달라 지는거지?

### build.gradle 설정 비교
```groovy
plugins {
    ...
    id "com.ewerk.gradle.plugins.querydsl" version "1.0.10"
}

compileQuerydsl{
    options.annotationProcessorPath = configurations.querydsl
}

configurations {
    ...
    querydsl.extendsFrom compileClasspath
}

def querydslSrcDir = 'src/querydsl/generated'

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

project.afterEvaluate {
    project.tasks.compileQuerydsl.options.compilerArgs = [
            "-proc:only",
            "-processor", project.querydsl.processors() +
                    ',lombok.launch.AnnotationProcessorHider$AnnotationProcessor'
    ]
}

dependencies {
    implementation("com.querydsl:querydsl-jpa") // querydsl
    implementation("com.querydsl:querydsl-apt") // querydsl
    ...
}
```

- querydsl plugin 방식과 비교를 해보면은 많은 설정 값이 추가되는 것을 확인할 수 있다.
- lombok을 사용하고 있어서 더 많은 설정이 추가된 부분이 있지만, 대부분 lombok을 사용한다고 가정하면 앞에 
방식과 비교하여 상당히 많은 설정이 추가가 발생한다.

### compileQuerydsl 시점
- querydsl plugin을 사용하게 되면, querydsl을 생성하는 방법은 gradle option에 있는
compileQuerydsl 사용하거나, build를 통해서 compile이 되어야 생성이 된다.
- 해당 방식을 이용하게 되면, compileJava 시점에 querydsl QModel이 생성이된다
- 즉, intellij에 Run, Debug를 통해서 서버를 실행할 때도 자동으로 최신 querydsl QModel을 생성하게 된다.

### Cache 유무
- gradle의 cache를 사용하기 위해서는 `org.gradle.caching=true` 설정을 해주어야 한다.
- querydsl plugin을 이용해서도 cache가 가능한지는 모르겠지만, annotationProcessor 방식을 이용하면 Cache가 가능하다. 
- 그 이유는 compileJava 부분에 QModel을 생성하기에 변경이 발생하지 않는다면 gradle에서
 증분 컴파일 옵션에 의해서 cache key를 사용할 수 있게 된다.

## 마치며
- cache를 타기 위해서는 같은 부분에 변경이 없어야지만 cache를 이용하기에 무조건 좋아지는 것은 아니다.
- 또한, build시에 gradle cache key를 매번 변경시키는 로직이 포함되어 있다면, cache를 이용하지 못 한다.
(ex: download files, springboot-buildInfo etc)
- querydsl plugin은 더 이상 개발이 되고 있는지 모르겠고.. gradle에서 deprecated된 compile을 사용하고 있어
추후 gradle version을 올릴때 제약사항이 될꺼 같아 변경하는 것이 좋아보인다.

- - - 
[build.gradle](https://github.com/KangWooJin/spring-study/blob/master/querydsl/build.gradle)
관련 example code는 github에 올려두었으니 참고~!
 