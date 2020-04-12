---
title: "[Spring] Gradle build 최적화 하기" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-04-12 12:00:00+09:00 
excerpt: Gradle을 사용할 때 build를 빠르게 하기 위한 셋팅을 적용해보자.
---

## 들어가며
Spring을 사용하면은 Maven, Gradle로 나뉘게 되는데 Gradle을 사용했을 때 
build 성능을 최적화 하는 방법에 대해서 알아보자.

## gradle.properties
gradle에 config를 변경하기 위해서는 해당 프로젝트 root에 `gradle.properties`를 추가 해준 뒤
아래 옵션을 추가 해준다.

```groovy
org.gradle.daemon=true
org.gradle.parallel=true
org.gradle.caching=true
```

## daemon
gradle을 빌드시에 gradle daemon에 의해서 빌드가 되는데 기본 옵션은 true로 되어 있어서 굳이 할 셋팅할 필요는 없다.

daemon으로 background로 동작하게 되어 약 3시간 정도 떠있다가 자연스럽게 없어진다.

gradle daemon이 떠 있는지 확인하려면 아래 명령어로 확인이 가능하다.
 
```groovy
./gradlew --status 
```


```groovy
   PID STATUS   INFO
 84502 IDLE     5.0

Only Daemons for the current Gradle version are displayed. See https://docs.gradle.org/5.0/userguide/gradle_daemon.html#sec:status
```

## parallel
프로젝트를 구성할 때 MSA로 구성하던지 multi module 방식으로 구성하게 된다.

만약 multi module로 구성되어 있을 때 build를 하게 되면 순차적으로 모듈을 빌드하게 된다.

이럴 때 parallel 옵션을 추가되면 각 module간에 병렬로 진행할 수 있는 module과 task들은
병렬로 진행하게 되어 빌드를 하는데 있어 더욱 빠르게 진행할 수 있게 된다.


![sequence](/assets/images/spring/gradle-build-sequence.png)
- parallel 설정을 주지 않은 상태

![parallel](/assets/images/spring/gradle-build-parallel.png)
- parallel 설정을 준 상태

## caching

caching 설정을 주게 되면 local에 저장되어 있는 cache 파일을 비교하여 같은 version인 경우
해당 caching 파일을 사용하게 되어 성능적 이점을 가져올 수 있다.

![no-caching](/assets/images/spring/gradle-build-no-cahing.png)
- no-caching 상태

![caching](/assets/images/spring/gradle-build-caching.png)
- caching 상태

## 마치며
빌드가 왜 이렇게 느리게 될까 싶으면 gradle에 Profile을 활용 하여
 어디서 병목 현상이 발생하고 있는지
확인이 가능하다

그리고 `--scan`을 붙이게 되면 gradle에서 web ui로 더 친절하게 제공해주는 부분도 있어서 쉽게 확인 할 수 있었다.

daemon, caching, parallel에 대해서 지식은 없고 좋다라는 것만 알고 있었는데 간단하게라도
공부한걸 기록해둔다.

[https://guides.gradle.org/performance/](https://guides.gradle.org/performance/) 

- - - 
