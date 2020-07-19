---
title: "[Spring] Gradle build 최적화 하기 (2)" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-07-19 12:00:00+09:00 
excerpt: Gradle을 사용할 때 build를 빠르게 하기 위한 셋팅을 적용해보자.
---

## 들어가며
예전에 한번 gradle 튜닝 관련해서 글을 쓴적이 있는데, 좀 더 빠르게 하기 위한 글을 작성해두려고 한다.

해당 방식은 앞에 작성한 글에 적용된 옵션이 다 적용되어 있어야 한다.


[Gradle Build Performance]({% post_url spring/2020-04-12-gradle-build-performance %}) 글 보러가기

## Running tests

- gradle에 테스트 관련해서 설정을 변경하면 build시에 성능 향상을 가져올 수 있다.
 
### Parallel Testing
```groovy
tasks.withType(Test) {
    maxParallelForks = 4
}

or

tasks.withType(Test) {
    maxParallelForks = Runtime.runtime.availableProcessors().intdiv(2) ?: 1
}
```

- 병렬로 테스트를 실행하는 방법을 추가하여 테스트를 빠르게 효율적으로 동작 시킨다.
- 단, 이 방식을 사용하는 경우 병렬로 테스팅이 가능하게 테스트가 짜여져 있어야 한다.
- 공식적으로 추천하는 방식은 이용 가능한 CPU에 반을 사용하는 방법이다.
- 전체 테스트를 프로세스가 나눠서 가져가기에, 병목 현상이 있는 테스트가 있는 경우 비슷한 
성능을 가져올 수 있으니 참고!

### Forking options
```groovy
tasks.withType(Test) {
    forkEvery = 100
}
```

- 해당 부분은 속도를 빠르게 한다기 보다, 안정성을 높이는 설정이다.
- 테스트를 하다보면 out of memory가 발생할 수 있는데, forkEvery의 수 만큼 테스트를 진행하고
새로운 프로세스를 fork하여 이어서 진행하는 방식이다.
- fork를 해서 진행하다보니, fork가 발생하는 횟수 만큼 느려질 수 밖에 없지만, 중간에 out of memory를
회피할 수 있는 방법 중 한 가지이다.
- out of memory를 회피하기 위해서는 heap size를 늘려야 하는데 heap size를 늘리면 garbage collection
비용이 많이 발생하게 된다.
- fork vs garbage collection 중 fork 방식이 더 비용이 적은 방식이라, memory를 늘리는 방식 보다는
적당한 forkEvery를 계산하여 설정하는게 좋다.
- forkEvery는 VM을 이용해서 해당 값이 작으면 비용이 많이 드는 작업이기에, 유의해야 한다.

### Report generation
```groovy
tasks.withType(Test) {
    reports.html.enabled = false
    reports.junitXml.enabled = false
}
```

- 테스트 시에 테스트 결과 리포트를 생성하지 않도록 하여 성능을 향상 시킨다.
 
### Compiler daemon
```groovy
tasks.withType(JavaCompile) {
    options.fork = true
}
```

- Gradle Java 플러그인에 JavaCompile을 사용하는 경우, 별도의 프로세스로 실행할 수 있도록 한다.
- 별로의 프로세스는 빌드 기간 재사용될 수 있어, fork의 오버헤드를 최소하하고, 메인 gradle의 데몬의
 garbage collection을 줄여준다.
- 단, 해당 옵션은 gradle을 parallel로 설정이 되어 있어야 동작한다. (default가 parallel) 
 
## Incremental compilation
```groovy
tasks.withType(JavaCompile) {
    options.incremental = true
}
```

- gradle은 변경의 영향을 받은 클래스만 다시 컴파일하여, 변경이 발생하지 않은 부분은 skip 처리 한다.
- gradle 4.10 이후부터는 해당 설정이 true가 되었다, 만약 4.10 이전 버전인 경우 해당 옵션을 추가해야 한다.

## 마치며
- query dsl을 사용하는 유저들 중 [query dsl plugin](https://plugins.gradle.org/plugin/com.ewerk.gradle.plugins.querydsl)을 사용하게 되면,
querydsl이 cache가 되지 않아, 매번 생성하는 불편함이 있다.
- cache가 될 수도 있는데... 어떻게 하는지 찾지를 못 하였다.
- gradle에서 querydsl을 캐시하여 build 속도 향상을 하는 방법에 대해서는 다음 포스트에 추가!
- gradle에서 공식적으로 튜닝하는 방법에 대해서는 [Gradle Performance](https://guides.gradle.org/performance/#apply_plugins_judiciously)
에 나와 있으니 참고하면 된다.
 

- - - 
