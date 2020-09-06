---
title: "[Spring Security] Spring Security Basic - AnonymousAuthenticationFilter" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-09-06 18:00:00+09:00
excerpt: AnonymousAuthenticationFilter가 언제 생성되고 무슨 역할을 하는지 알아보자.
---

## 들어가며
AnonymousAuthenticationFilter가 언제 생성되고 무슨 역할을 하는지 알아보자.

## 언제 생성 되는가?

```java
@Override
public void init(H http) {
  if (authenticationProvider == null) {
    authenticationProvider = new AnonymousAuthenticationProvider(getKey());
  }
  if (authenticationFilter == null) {
    authenticationFilter = new AnonymousAuthenticationFilter(getKey(), principal,
				authorities);
  }
  authenticationProvider = postProcess(authenticationProvider);
  http.authenticationProvider(authenticationProvider);
}
```

- `AnonymousConfigurer`에서 init 부분에서 `AnonymousAuthenticationFilter`가 생성이 된다.
- `AnonymousAuthenticationFilter`가 생성될 때 기본으로 셋팅되는 값들이 존재한다.
- key는 UUID로 random으로 생성을 하고 principal은 `anonymousUser`, authorities는 `ROLE_ANONYMOUS`로 기본값이 생성 된다.
- key는 `AnonymousAuthenticationFilter`로 생성된 token인지 확인을 하기 위한 용도로 사용 된다.

## 무슨 역할을 하는가?
- `SeucotyContext`에 Authentication이 null인 경우 `AnonymousAuthenticationToken`을 주입해주는 역할을 담당한다.
- `SecurityContext`에 인증이 성공한 경우 `Authentication`이 `ThreadLocal`을 사용하여 저장이 되는 것을 [이전 포스트]({% post_url spring-security/2020-08-06-spring-security-basic-security-context-holder %})에서 확인하였다.


```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
		throws IOException, ServletException {

  if (SecurityContextHolder.getContext().getAuthentication() == null) {
    SecurityContextHolder.getContext().setAuthentication(
      createAuthentication((HttpServletRequest) req));

    if (logger.isDebugEnabled()) {
      logger.debug("Populated SecurityContextHolder with anonymous token: '"
					+ SecurityContextHolder.getContext().getAuthentication() + "'");
    }
  }
  else {
    if (logger.isDebugEnabled()) {
      logger.debug("SecurityContextHolder not populated with anonymous token, as it already contained: '"
					+ SecurityContextHolder.getContext().getAuthentication() + "'");
    }
  }

  chain.doFilter(req, res);
}
```
- `Authentication`이 null 인경우 `SecurityContextHolder`에 `SecurityContext`에 Authentication에 값을
채워주는 부분을 한다.
- 만약 null이 아닌 경우 다음 필터로 진행 한다.

```java
protected Authentication createAuthentication(HttpServletRequest request) {
  AnonymousAuthenticationToken auth = new AnonymousAuthenticationToken(key,
			principal, authorities);
  auth.setDetails(authenticationDetailsSource.buildDetails(request));

  return auth;
}
```
- `createAuthentication`에서 `AnonymousAuthenticationToken`을 생성한다.
- `AnonymousAuthenticationToken`을 생성하는 이유는 SecurityFilter에서 Authentication이 null인지 확인하는 부분을 없애고, null class인 `AnonymousAuthenticationToken`을 이용해서
null인지 확인하게 된다.

```java
private Class<? extends Authentication> anonymousClass = AnonymousAuthenticationToken.class;

public boolean isAnonymous(Authentication authentication) {
  if ((anonymousClass == null) || (authentication == null)) {
    return false;
  }

  return anonymousClass.isAssignableFrom(authentication.getClass());
}
```

- `AuthenticationTrustResolverImpl.isAnonymous` 부분을 보게 되면, `Authentication`이
 `AnonymousAuthenticationToken`인지 확인하는 부분이 존재한다.
- 이런식으로 null인지 아닌지를 확인하는 것이 아닌, SpringSecurity에서는 null object를 이용해서 처리 하도록 설계가 된거 같다.
- [null object pattern](https://en.wikipedia.org/wiki/Null_object_pattern) 참고


