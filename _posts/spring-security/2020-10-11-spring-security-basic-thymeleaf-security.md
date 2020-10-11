---
title: "[Spring Security] Spring Security Basic - Thymeleaf security" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-10-11 12:00:00+09:00
excerpt: Spring security와 thymeleaf를 이용할 때 어떻게 사용할 수 있는 알아보자.
---

## 들어가며
Spring security를 이용할 때 view를 제공할 수 있는 라이브러리는 다양하게 존재한다. 그 중에서 thymeleaf와 spring security를
연동해서 사용하는 방법에 대해서 알아보자.

기본적으로 Spring Security의 기본적인 셋팅이 되어 있다고 가정하고, thymeleaf에 문법에 대해서만 어느정도 알아 본다.

## 의존성

```java
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-security'
	implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
	implementation 'org.springframework.boot:spring-boot-starter-web'
	// thymeleaf extras
	implementation 'org.thymeleaf.extras:thymeleaf-extras-springsecurity5'
    ...
}
```
- spring security, thymeleaf, web은 필수로 필요하다.
- `thymeleaf-extras`가 view에서 spring security를 사용할 수 있도록 해주는 라이브러리 이다.

## 어떻게 사용하나?

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://thymeleaf.org">
<head>
  <meta charset="UTF-8">
  <title>Index</title>
</head>
<body>
  <h1 th:text="${message}">Hello</h1>
  <div th:if="${#authorization.expr('isAuthenticated()')}">
    <h2 th:text="${#authentication.name}">Name</h2>
    <a href="/logout" th:href="@{/logout}">Logout</a>
  </div>

  <div th:unless="${#authorization.expr('isAuthenticated()')}">
    <a href="/login" th:href="@{/login}">Login</a>
  </div>
</body>
</html>
```

- thymeleaf-extras 라이브러리를 추가하게 되면 `${authorization}`, `${authentication}` 처럼 인증, 인가에 대한 데이터를 불러올 수 있다.
- 서버에서 ModelAndView를 통해서 데이터를 전달해야 view에서 사용가능하지만, 해당 라이브러리를 이용하면 쉽게 사용이 가능하다.
- 어느정도 자동완성을 제공해주긴하나, 오타가 발생하는 경우 찾기가 어렵기에 오타가 생기지 않도록 주의 해야 한다.

## 네임스페이스 방법
```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://thymeleaf.org" xmlns:sec="http://thymeleaf.org/extras/spring-security">
<head>
  <meta charset="UTF-8">
  <title>Index</title>
</head>
<body>
  <h1 th:text="${message}">Hello</h1>
  <div sec:authorize-expr="isAuthenticated()">
    <h2 sec:authentication="name">Name</h2>
    <a href="/logout" th:href="@{/logout}">Logout</a>
  </div>

  <div sec:authorize-expr="!isAuthenticated()">
    <a href="/login" th:href="@{/login}">Login</a>
  </div>
</body>
</html>
```

- 앞에 방식보다 조금 더 편리한 방법은 네임스페이스를 이용해서 자동완성을 이용하는 방법이 있다.
- `xmlns:sec="http://thymeleaf.org/extras/spring-security"`를 추가하여 네임스페이스를 추가하게 되면 좀 더 편리하게 자동완성을 제공 해 준다.
- `sec:authorize-expr="isAuthenticated()"`, `sec:authentication="name"` 처럼 쉽게 사용이 가능하다.
- 이전 방식을 이용하면 입력에 대해서 오타가 있는 경우 parse error만 생기지, 어디가 잘 못 되었는지 정확 하게 집어 주지 않아서 주의 깊게 다시 확인 해야 한다.

## 마치며
- 여러가지 view 라이브러리를 사용해보았지만, thymeleaf는 한번만 사용해보았다.. 다른 vue.js, react에 비해서 부족한 부분이 많아 보였다.
- security를 사용하더라도, thymeleaf를 사용하는 경우는 적지 않을까 싶다...