---
title: "[Spring Security] Spring Security Basic - SecurityContextPersistenceFilter" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-16 19:00:00+09:00
excerpt: SecurityContextPersistenceFilter가 무엇을 하는지, 어떤 방식으로 동작하는지 알아 보자.
---

## 들어가며

`SecurityContextPersistenceFilter`가 무엇을 하는지, 어떤 방식으로 동작하는지 알아 보자.

## SecurityContextPersistenceFilter

- `SecurityContextPersistenceFilter`는 `SecurityContext`의 값을 Session에 저장해 두고 
다음 요청에서도 재활용할 수 있게 하는 역할을 담당하고 있다.

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
  ...
  HttpRequestResponseHolder holder = new HttpRequestResponseHolder(request,
			response);
  SecurityContext contextBeforeChainExecution = repo.loadContext(holder);

  try {
    SecurityContextHolder.setContext(contextBeforeChainExecution);

    chain.doFilter(holder.getRequest(), holder.getResponse());

  }
  finally {
    SecurityContext contextAfterChainExecution = SecurityContextHolder
				.getContext();
    SecurityContextHolder.clearContext();
    repo.saveContext(contextAfterChainExecution, holder.getRequest(),
			holder.getResponse());
    request.removeAttribute(FILTER_APPLIED);
    if (debug) {
      logger.debug("SecurityContextHolder now cleared, as request processing completed");
    }
  }
}
```

- `repo.loadContext` 부분에서 Session에 있는 `SecurityContext`을 불러와서 `SecurityContextHolder`에 값을
셋팅 한다.
- finally 부분에서 사용되었던 `SecurityContext`를 다시 `repo.saveContext`를 통해서 Session에 저장을 한다.
- 해당 Filter의 역할을 보면, 언제 동작되어야 정상적인지 알 수 있는게, 인증이 되기 전에 실행 되어야 인증 전에 이전에 인증되었던 
결과를 확인하여 인증을 스킵할 수 있고, 인증 후에 결과를 저장할 수 있다.
- Session을 이용하는 방식은 Default 방식이고, 필요에 따라서 Session 말고 다른 곳에 저장을 할 수 있다.

### SecurityContextRepository

```java
public interface SecurityContextRepository {
  SecurityContext loadContext(HttpRequestResponseHolder requestResponseHolder);
  void saveContext(SecurityContext context, HttpServletRequest request,
			HttpServletResponse response);

  boolean containsContext(HttpServletRequest request);
}
```

- `SecurityContextPersistenceFilter`에서 실제로 `SecurityContext`를 저장하는 역할을 담당하고 있다.
- default는 `HttpSessionSecurityContextRepository`으로 생성이 되며, 해당 부분을 custom하게 하려면, 
http쪽 configure할 때 setSharedObject에 custom `SecurityContextRepository`를 할당해 주면 된다.

### SessionManagementConfigurer

```java
public void init(H http) {
  SecurityContextRepository securityContextRepository = http
			.getSharedObject(SecurityContextRepository.class);
  boolean stateless = isStateless();

  if (securityContextRepository == null) {
    if (stateless) {
      http.setSharedObject(SecurityContextRepository.class,
					new NullSecurityContextRepository());
    }
    else {
      HttpSessionSecurityContextRepository httpSecurityRepository = new HttpSessionSecurityContextRepository();
      httpSecurityRepository
					.setDisableUrlRewriting(!this.enableSessionUrlRewriting);
      httpSecurityRepository.setAllowSessionCreation(isAllowSessionCreation());
      AuthenticationTrustResolver trustResolver = http
					.getSharedObject(AuthenticationTrustResolver.class);
      if (trustResolver != null) {
        httpSecurityRepository.setTrustResolver(trustResolver);
      }
      http.setSharedObject(SecurityContextRepository.class,
					httpSecurityRepository);
    }
  }
  RequestCache requestCache = http.getSharedObject(RequestCache.class);
  if (requestCache == null) {
    if (stateless) {
      http.setSharedObject(RequestCache.class, new NullRequestCache());
    }
  }
  http.setSharedObject(SessionAuthenticationStrategy.class,
			getSessionAuthenticationStrategy(http));
  http.setSharedObject(InvalidSessionStrategy.class, getInvalidSessionStrategy());
}
```

- `SessionManagementConfigurer.init()`에서 `securityContextRepository` 값을 할당하게 되어 있다.
- 기본적으로 `HttpSecurity`에 `SecurityContextRepository`를 미리 할당 하지 않으면,
`SessionCreationPolicy` 값을 확인하여, `NullSecurityContextRepository`, `HttpSessionSecurityContextRepository`를 할당하게 되어 있다.

## 마치며
- `SecurityContextPersistenceFilter`에 `SecurityContextRepository`를 잘 활용한다면, 원하는 방식으로
`SecurityContext`를 저장할 수 있다.
  