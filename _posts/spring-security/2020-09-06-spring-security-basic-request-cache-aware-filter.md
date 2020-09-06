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
- RequestCacheConfigurer가 configure 될 때 RequestCache를 생성 한다.
- 좀 더 자세하게 설명하자면, SecurityFilterChain이 생성될 때이다.
- 이 때, RequestCache의 생성 우선순위가 존재 한다.

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

