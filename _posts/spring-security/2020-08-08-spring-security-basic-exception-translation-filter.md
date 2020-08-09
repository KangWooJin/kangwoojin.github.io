---
title: "[Spring Security] Spring Security Basic - ExceptionTranslationFilter" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-09 12:00:00+09:00
excerpt: FilterSecurityInterceptor에서 발생하는 Exception이 어떻게 처리되는지 알아 보자. 
---

## 들어가며
[FilterSecurityInterceptor]({% post_url spring-security/2020-08-08-spring-security-basic-access-decision-manager %})에서
`AccessDeniedException`, `AuthenticationException`에 에러가 발생하는데, 해당 에러가 발생할 경우 어떻게 처리가 되는지 알아 보자.

## AccessDeniedException 처리 과정

```java
public interface AccessDecisionManager {
	void decide(Authentication authentication, Object object,
			Collection<ConfigAttribute> configAttributes) throws AccessDeniedException,
			InsufficientAuthenticationException;
}
``` 

- `AccessDeniedException`이 발생 하는 조건은 `FilterSecurityInterceptor`를 통해서 `AccessDecisionManager`에 
Authority를 확인 하다가 발생하게 된다.


```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
  try {
    chain.doFilter(request, response);
    logger.debug("Chain processed normally");
  }
  catch (IOException ex) {
    throw ex;
  }
  catch (Exception ex) {
    // 여기서 catch
    Throwable[] causeChain = throwableAnalyzer.determineCauseChain(ex);
	RuntimeException ase = (AuthenticationException) throwableAnalyzer
			.getFirstThrowableOfType(AuthenticationException.class, causeChain);

	if (ase == null) {
	  ase = (AccessDeniedException) throwableAnalyzer.getFirstThrowableOfType(
	    AccessDeniedException.class, causeChain);
	}

	if (ase != null) {
	  if (response.isCommitted()) {
	    throw new ServletException("Unable to handle the Spring Security Exception because the response is already committed.", ex);
	  }
	  handleSpringSecurityException(request, response, chain, ase);
	}
	else {
	  // Rethrow ServletExceptions and RuntimeExceptions as-is
	  if (ex instanceof ServletException) {
	    throw (ServletException) ex;
	  }
	  else if (ex instanceof RuntimeException) {
	    throw (RuntimeException) ex;
	  }
	  throw new RuntimeException(ex);
	}
  }
}
```

- `decide`를 하다 `AccessDeniedException`이 발생하게 되면 `ExceptionTranslationFilter`에서 catch하게 된다.

![exception-translation-filter](/assets/images/spring-security/exception-translation-filter.png)

- `ExceptionTranslationFilter`는 `FilterSecurityInterceptor` Filter 이전에 동작하게 되어 있고 chain 형태로
 filter가 동작하기에 `FilterSecurityInterceptor` error를 throw하게 되면, `ExceptionTranslationFilter`에서 catch 된다.

```java
private void handleSpringSecurityException(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain, RuntimeException exception)
			throws IOException, ServletException {
  if (exception instanceof AuthenticationException) {
    logger.debug(
      "Authentication exception occurred; redirecting to authentication entry point",
				exception);

    sendStartAuthentication(request, response, chain,
				(AuthenticationException) exception);
  }
  else if (exception instanceof AccessDeniedException) {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    if (authenticationTrustResolver.isAnonymous(authentication) || authenticationTrustResolver.isRememberMe(authentication)) {
      logger.debug(
        "Access is denied (user is " + (authenticationTrustResolver.isAnonymous(authentication) ? "anonymous" : "not fully authenticated") + "); redirecting to authentication entry point",
					exception);

      sendStartAuthentication(
						request,
						response,
						chain,
						new InsufficientAuthenticationException(
							messages.getMessage(
								"ExceptionTranslationFilter.insufficientAuthentication",
								"Full authentication is required to access this resource")));
    }
    else {
      logger.debug(
        "Access is denied (user is not anonymous); delegating to AccessDeniedHandler",
					exception);

      accessDeniedHandler.handle(request, response,
					(AccessDeniedException) exception);
    }
  }
}
```
- `ExceptionTranslationFilter`에서 실제로 에러를 핸들링 하는 부분은 `handleSpringSecurityException`이다.
- `handleSpringSecurityException`에서 `AuthenticationException`인 경우 바로 `sendStartAuthentication` 추가 인증을 실시 한다.
- `AccessDeniedException`라면, token이 AnonymousToken인 경우 `sendStartAuthentication` 추가 인증을 하고,
그게 아닌 경우 `accessDeniedHandler`를 통해서 `AccessDeniedException`을 처리한다.

```java
protected void sendStartAuthentication(HttpServletRequest request,
		HttpServletResponse response, FilterChain chain,
		AuthenticationException reason) throws ServletException, IOException {
  // SEC-112: Clear the SecurityContextHolder's Authentication, as the
  // existing Authentication is no longer considered valid
  SecurityContextHolder.getContext().setAuthentication(null);
  requestCache.saveRequest(request, response);
  logger.debug("Calling Authentication entry point.");
  authenticationEntryPoint.commence(request, response, reason);
}
```

- `sendStartAuthentication`에서 `authenticationEntryPoint`

### AuthenticationEntryPoint

```java
public interface AuthenticationEntryPoint {
	void commence(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException authException) throws IOException, ServletException;
}
```

- `ExceptionTranslationFilter`에서 사용되는 추가 인증을 할때 사용되는 인터페이스이다.
- `sendStartAuthentication`에서는 `authenticationEntryPoint.commence`를 통해서 추가 인증을 실시 하게 된다.

```java
public void commence(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException authException) throws IOException, ServletException {
  String redirectUrl = null;
    ...
  else {
	redirectUrl = buildRedirectUrlToLoginPage(request, response, authException);
  }
  redirectStrategy.sendRedirect(request, response, redirectUrl);
}
```

- `WebSecurityConfigurerAdapter`에서 formLogin을 추가 해둔 경우 `AccessDeniedException`이 발생하게 되면,
LoginUrlAuthenticationEntryPoint`에서 핸들링되고, `/login` page로 redirect 된다.

## AuthenticationException 처리 과정

- `AuthenticationException`이 발생할려면, `AuthenticationEntryPoint`에서 exception이 발생하거나,
항상 인증모드를 해서 `FilterSecurityInterceptor`에서 authentication을 하는 케이스에서마 발생 한다.
- `AuthenticationException` 방식도 `AccessDeniedException`와 비슷하게 `AuthenticationEntryPoint`에서
추가 인증을 시도하고 성공하면 넘어가고 실패하면 `AuthenticationException`이 발생하고 종료하게 된다.


## 마치며
- `AuthenticationException`과 `AccessDeniedException`이 발생했을 때 어디서 처리 되는지 알아보았다.
- `ExceptionTranslationFilter`에서 `AuthenticationException`를 처리 한다고 되어 있는데, Filter의 순서를 보면
`FilterSecurityInterceptor`에서 발생한 `AuthenticationException` 만 처리하고, 다른 곳에서 발생한 `AuthenticationException`은
처리가 되지 않으니 오해 하지 말아야 한다.

