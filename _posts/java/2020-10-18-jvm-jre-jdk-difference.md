---
title: "[Java] JDK, JRE, JVM 차이" 
categories:
  - Programing
tags:
  - Java
toc: true
toc_sticky: true
date: 2020-10-18 14:00:00+09:00 
excerpt: JDK, JRE, JVM에 차이에 대해서 알아보자.
---

## 들어가며
JDK, JRE, JVM에 차이에 대해서 알아보자.

## JDK, JRE, JVM
![jdk-jre-jvm](/assets/images/java/jdk-jre-jvm.png)

### JVM(Java virtual Machine)
- 소스코드(.java) 파일을 컴파일하여 생긴 자바 바이트 코드(.class)를 OS에 특화된 코드로 변환하여 실행해주는 도구이다.
- 바이트 코드를 실행하는 표준이자 구현체다.
- JVM 밴더: [오라클](https://www.oracle.com/kr/java/technologies/javase/javase-downloads.html), [아마존](https://aws.amazon.com/ko/corretto/), [Azul](https://www.azul.com/downloads/zulu-community/?architecture=x86-64-bit&package=jdk)
- JVM은 특정 플랫폼에 종속적이라 리눅스, 윈도우즈, Mac 환경에 JVM은 서로 다르게 되어 있다.
- 소스 코드를 작성할 때 사용하는 자바 언어는 플랫폼에 독립적이다.

### JRE(Java Runtime Environment)
- JVM, 코어 클래스 및 지원파일로 구성되어 있다.
- JVM이 자바 프로그램을 동작시킬 때 필요한 라이브러리 파일들을 갖고 있다.
- JRE에서는 자바컴파일러(javac)가 존재하지 않고 실행관련된 파일들만 갖고 있다.
- Java Application 을 실행할 수 있는 환경을 제공하는 설치 패키지이다.

### JDK(Java Development Kit)
- JRE + 개발에 필요한 툴(Development Tools)
- Development Tools : 개발하기 위한 용도
- JRE : 실행하기 위한 용도
- JDK : 개발자를 위한 용도

## 그 외

### 자바
- 단순 프로그래밍 언어로 플랫폼에 독립적이다.
- JDK에 들어있는 자바 컴파일러(javac)를 사용하여 바이트코드(.class 파일)로 사용할 수 있음

### JVM 언어
- JVM 기반 으로 동작하는 프로그래밍 언어
- 클로저, Groovy, JRuby, Jython, Kotlin, Scala

## 마치며
- Java 개발자라면 해당 3가지를 많이 들어봤을 것이다, 하지만 서로의 차이까지 몰랐다면 간단하게 알고 넘어가는 것도 좋아보인다!