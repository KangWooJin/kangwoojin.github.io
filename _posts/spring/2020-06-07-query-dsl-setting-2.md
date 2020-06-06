---
title: "[Spring] intellij에서 query dsl로 생성되는 folder가 sources에서 풀리는 현상을 해결" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-06-07 08:00:00+09:00 
last_modified_at: 2020-06-07 08:00:00
excerpt: intellij에서 query dsl로 생성되는 folder가 sources에서 풀리는 현상을 해결 해보자.
---

## 들어가며
예전에 [query dsl 관련 셋팅]({% post_url spring/2020-04-06-query-dsl-setting %})을 올려두었다.

해당 방식을 이용하면 gradle이 reimport가 되면 query dsl로 생성된 `src/main/generated`가 sources에서 풀리는 현상이 발생한다.

![build](/assets/images/spring/module-reimport.jpg)

>Module is imported from gradle. any changes made in its configuration may be lost after reimporting

추측되는 원인은 build 방식을 gradle로 설정했기에 intellij에서 변경이 발생하면 gradle로 설정된 부분이 풀리는 것 같다.
 
이러한 문제에 대해서 해결 방법을 찾아서 기록해 둔다.
 
## 필요 조건
해당 현상이 발생하는 필요 조건이 있다.

![build](/assets/images/spring/query-dsl-build.jpg)

1. intellij를 이용한다.
2. build, Execution, Deployment -> Build tools -> gradle에서 Build and run unsing이 gradle이어야 한다.
 
>intellij의 버전이 낮은 경우 default가 gradle로만 되어 있어 선택할 수 있는 부분이 없다.

## 문제 현상
![notFoundQModel](/assets/images/spring/not-found-qmodel.jpg) 

문제 없이 query dsl을 잘 쓰고 있다가 작업 중인 branch에서 다른 branch로 바꾸거나 

gradle을 reimport하는 경우 해당 이미지 처럼 QModel이 빨간줄로 인식되는 문제가 종종 발생하였다.

## 해결 방법
![module-sources](/assets/images/spring/module-sources.jpg)
`Project Structure`에서 query dsl로 생성해둔 folder가 sources로 되어 있지 않다면 추가 해준다.

엄청 간단하게 해결이 가능하다! 그렇지만 매번 추가 해주기가 귀찮다!

어떻게 하면 자동으로 셋팅이 될 수 있을까 고민하게 되었다.

## 자동 셋팅 방법
```groovy
def querydslSrcDir = 'src/main/generated'

sourceSets {
    main {
        java {
            srcDirs = ['src/main/java', querydslSrcDir]
        }
    }
}
```

query dsl을 셋팅 할 때 `sourcesSets`을 통해서 sources가 자동으로 추가 되는 설정을 이미 추가를 해두었다.

그렇지만 `src/main/java`와 `src/main/generated`로 `src/main`이라는 같은 경로를 사용하고 있는게 문제 인거 같다.

query dsl로 생성되는 folder의 경로를 `src/querydsl/generated`로 변경하니 문제가 해결 되었다.

## 마치며
사소한 부분이었지만 귀찮은 일을 하기가 싫으니 자동화를 하게 되었다.

해당 방식 말고도 build시에 `gradle`이 아닌 `intellij`로 설정을 하게 되면 문제가 해결이 된다.

그렇지만 intellij의 버전이 낮은 분들은 해결이 안되며,
 
gradle의 추가적인 설정에 따라서 정상 동작하지 않을 수 있다.

- - - 
[querydsl](https://github.com/KangWooJin/spring-study/tree/master/querydsl)
관련 example code는 github에 올려두었으니 참고~!
 