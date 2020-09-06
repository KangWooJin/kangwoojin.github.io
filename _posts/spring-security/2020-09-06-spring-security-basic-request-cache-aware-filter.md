---
title: "[Spring Security] Spring Security Basic - RequestCacheAwareFilter" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-09-06 12:00:00+09:00
excerpt: RequestCacheAwareFilter가 언제 생성되고 무슨 역할을 하는지 알아보자.
---

## 들어가며
RequestCacheAwareFilter가 언제 생성되고 무슨 역할을 하는지 알아보자.

## 언제 생성 되는가?

```java
@Override
public void configure(H http) {
  RequestCache requestCache = getRequestCache(http);
  RequestCacheAwareFilter requestCacheFilter = new RequestCacheAwareFilter(
    requestCache);
  requestCacheFilter = postProcess(requestCacheFilter);
  http.addFilter(requestCacheFilter);
}
```

```java
private RequestCache getRequestCache(H http) {
  RequestCache result = http.getSharedObject(RequestCache.class);
  if (result != null) {
    return result;
  }
  result = getBeanOrNull(RequestCache.class);
  if (result != null) {
    return result;
  }
  HttpSessionRequestCache defaultCache = new HttpSessionRequestCache();
  defaultCache.setRequestMatcher(createDefaultSavedRequestMatcher(http));
  return defaultCache;
}
```

- RequestCacheConfigurer가 configure 될 때 `RequestCacheAwareFilter`를 생성 한다.
- configure 부분을 좀 더 자세하게 설명하자면, SecurityFilterChain이 생성될 때이다.
- 이 때, RequestCache의 생성은 우선순위가 존재 한다.

1. http에 RequestCache가 이미 존재한다면, 해당 RequestCache를 사용
2. Bean으로 등록되어 있는 RequestCache를 사용
3. Default인 HttpSessionRequestCache를 사용

## 무슨 역할을 하는가?
- 현재 요청을 처리할 때 캐시되어 있는 요청이 있는 경우 캐시된 요청을 처리해주는 필터 이다.
- 말로만 설명하면 이해하기가 어려워 코드를 통해서 확인 해보자.

```java
public class RequestCacheAwareFilter extends GenericFilterBean {
  private RequestCache requestCache;

  public RequestCacheAwareFilter() {
    this(new HttpSessionRequestCache());
  }

  public RequestCacheAwareFilter(RequestCache requestCache) {
      Assert.notNull(requestCache, "requestCache cannot be null");
      this.requestCache = requestCache;
  }

  public void doFilter(ServletRequest request, ServletResponse response,
		FilterChain chain) throws IOException, ServletException {

    HttpServletRequest wrappedSavedRequest = requestCache.getMatchingRequest(
      (HttpServletRequest) request, (HttpServletResponse) response);

    chain.doFilter(wrappedSavedRequest == null ? request : wrappedSavedRequest,
			response);
  }

}
```

- `doFilter` 부분에서 `requestCache.getMatchingRequest`를 통해서 session에 저장되어 있는 request를 불러 온다.
- 만약 캐시된 요청이 존재한다면, 캐시된 `wrappedSavedRequest`를 처리하고, 아닌 경우 `request`를 처리 한다.
- 좀 더 예시를 들자면, 인증이 필요한 웹사이트에 접근했을 때 로그인창으로 리다이렉트 된 뒤 로그인을 하면 로그인 하기 이전에 페이지로
돌아 가는 것을 종종 볼 수 있다.
- 로그인 하기전 요청을 캐시하고 있다가 로그인에 성공하는 경우 캐시된 요청을 로그인 이후에 처리해 주는 역할을 해주는게 `RequestCacheAwareFilter` 이다.

### 언제 저장 되는가?
- `Authorization`에 실패했을 때 발생되는 `AuthenticationException`, `AccessDeniedException`을 처리할 때 request를 저장하게 된다.
- `Authorization`은 `FilterSecurityInterceptor`에서 하게 되는데, 해당 필터에서 `Authorization`에러가 발생되는 경우 `ExceptionTranslationFilter`에서
exception을 처리하게 된다.

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
        request, response, chain, new InsufficientAuthenticationException( messages.getMessage(
					    "ExceptionTranslationFilter.insufficientAuthentication",
							"Full authentication is required to access this resource")));
    }
    else {
      logger.debug("Access is denied (user is not anonymous); delegating to AccessDeniedHandler", exception);
      accessDeniedHandler.handle(request, response,
					(AccessDeniedException) exception);
    }
  }
}
```
- exception이 발생하면 `handleSpringSecurityException`에서 Exception을 확인 후 처리하게 된다.

```java
protected void sendStartAuthentication(HttpServletRequest request,
	HttpServletResponse response, FilterChain chain,
	AuthenticationException reason) throws ServletException, IOException {
  SecurityContextHolder.getContext().setAuthentication(null);
  requestCache.saveRequest(request, response);
  logger.debug("Calling Authentication entry point.");
  authenticationEntryPoint.commence(request, response, reason);
}
```

- 그 중에서 `sendStartAuthentication`에서 `requestCache.saveRequest`를 통해서 해당 request를 session에 저장하게 된다. 기본값이 session이다.
- `Authorization` 관련 exception을 처리할 때 저장을 하고 나중에 `Authentication`에 성공한 경우 저장한 request를 처리하는 방식이다.
