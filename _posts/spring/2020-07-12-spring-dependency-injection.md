---
title: "[Spring] Spring dependency injection 순서" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-07-12 08:00:00+09:00 
excerpt: Spring에서 DI를 사용할 때 먼저 주입되는 순서를 알아보자.
---

## 들어가며
Jpa에 EntityListener를 사용할 때 dependency injection이 되는걸 보고 
어떻게 되었는지 궁금증을 품게 되었다.

해당 부분에 대해서 설명하기에 앞서 Spring에 대해서 기본적인 지식이 없으면 이해하기 어렵기에
기본적인 지식부터 작성해보려고 한다.

Spring에서는 DI를 통해서 원하는 객체를 주입 해주고 있다, 방법은 생성자, 필드, Setter로 3가지이다.

해당 주입 과정에도 무엇이 먼저 되는지 알아본다.

## Dependency Injection 순서

### 테스트 코드

```java
@Service
public class MainService {
    private final ConstructorService constructorService;
    private SetterService setterService;
    @Autowired
    private AutowiredService autowiredService;

    public MainService(ConstructorService constructorService) {
        System.out.println("MainService Constructor");
        System.out.println("MainService.constructorService : " + constructorService);
        System.out.println("MainService.setterService : " + setterService);
        System.out.println("MainService.autowiredService : " + autowiredService);
        this.constructorService = constructorService;
    }

    @Autowired
    public void setSetterService(SetterService setterService) {
        System.out.println("MainService setSetterService");
        System.out.println("MainService.constructorService : " + constructorService);
        System.out.println("MainService.setterService : " + setterService);
        System.out.println("MainService.autowiredService : " + autowiredService);
        this.setterService = setterService;
    }
}
```  

- `MainService`는 `@Service` annotation을 갖고 있는 서비스를 하나 만든다.
- `MainService`에서 생성자, 필드, Setter 주입을 받는 3가지의 필드를 갖도록 한다.
- 참고로 생성자 주입시에 `@Autowired`를 추가 하지 않아도 spring 1.5부터 알아서 주입을 해준다.

### 실행 순서
```
AutowiredService Constructor
ConstructorService Constructor
MainService Constructor
MainService.constructorService : kangwoojin.github.io.querydsl.dependencyinjection.ConstructorService@7ec3faad
MainService.setterService : null
MainService.autowiredService : null
SetterService Constructor
MainService setSetterService
MainService.constructorService : kangwoojin.github.io.querydsl.dependencyinjection.ConstructorService@7ec3faad
MainService.setterService : kangwoojin.github.io.querydsl.dependencyinjection.SetterService@f28c3431
MainService.autowiredService : kangwoojin.github.io.querydsl.dependencyinjection.AutowiredService@9c9ddf8e
```

- 해당 실행 순서에 대해서 자세하게 알아보자.

#### AutowiredService Constructor
```java
@Service
public class AutowiredService {
    public AutowiredService() {
        System.out.println("AutowiredService Constructor");
    }
}
```

- `AutowiredService`의 경우 아무런 필드도 없는 Service로 만들어 두었다.
- 따라서 Spring에서 Bean으로 먼저 등록 가능한것에 대해서 미리 bean으로 등록을 해둔다.
- 추측이지만 정렬 기준이 있어, 'A'로 시작한 `AutowiredService`가 먼저 생성되었다.

#### ConstructorService Constructor

```java
@Service
public class ConstructorService {
    public ConstructorService() {
        System.out.println("ConstructorService Constructor");
    }
}
```

- `ConstructorService` 또한, `AutowiredService`와 같이 아무런 필드가 없게 만들어 두었다.
- 따라서 Spring에서 Bean으로 먼저 등록하였고, 'A'->'C'라서 미리 생성이 되었다.

#### MainService Constructor

- `MainService`가 `Bean`으로 등록 되기 위해서는 앞에 3개 field의 `Service`가 미리 `Bean`으로 등록 되어 있어야 한다.
- `AutowiredService`, `ConstructorService`의 경우 미리 Bean으로 생성해 두었으니 추가적으로 생성은 하지 않는다.
- `MainService`의 생성자에서는 `ConstructorService`의 값만 존재하고 나머지는 null이다.
- 그 이유는 Spring에서는 생성자를 먼저 만든 다음에 다른 작업들이 진행되기 때문이다.

#### SetterService Constructor

- `MainService`에서 `SetterService`를 Setter 주입하기 위해서 `SetterService`를 Bean으로 만드는 작업을 진행한다.

```java
@Service
public class SetterService {
    public SetterService() {
        System.out.println("SetterService Constructor");
    }
}
```

- `SetterService` 또한 앞에 Service들처럼 바로 Bean으로 등록 된다.
- `MainService`에 Setter 주입시에는, 모든 필드 값이 생성되어 지는 것을 확인 할 수 있다.
- 도대체 필드 주입은 언제 발생했길래, 생성자 주입시에는 없고, Setter 주입에는 있을까?

### 필드 주입 과정
- 필드 주입은 해당 객체의 생성자를 생성 한 다음에 필드 주입이 발생한다.
- 생성자 주입과 필드 주입 부분은 `AbstractAutowireCapableBeanFactory.doCreateBean`에서 확인이 가능하다.
- `doCreateBean` method 안에 `createBeanInstance`에서 생성자 주입이 발생한다.
- 그 후 `populateBean`에 `postProcessProperties`에서 `Autowired`로 되어 있는 필드 주입을 진행 후 `Setter` 주입을 한다.

## 참고
- [[Spring] Bean 생성시 필드주입 시점은?](https://namocom.tistory.com/772)

## 마치며
- Spring에 DI에 대해서 주입되는 과정이 어려워서 알려고 하지 않은 부분도 있었는데,
IDE에 debug이 잘 되어 있어서 쉽게 순서를 확인할 수 있었다.
- 기본적인 부분에 대해서도 아직 잘 모르고 있어서.. 더 많이 노력해야 겠다고 느꼈다. ㅠㅠ 
  

- - -  