---
title: "[Spring Security] Spring Security Basic - CustomFilter 추가하기" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-09-20 14:00:00+09:00
excerpt: SpringSecurity에서 Custom Filter를 추가하는 방법에 대해서 알아보자.
---

## 들어가며
SpringSecurity에서 Custom Filter를 추가하는 방법에 대해서 알아보자.

## Custom Filter 추가하는 방법

### addFilter

```java
http.addFilter(new LoggingFilter());
```
- 해당 기능은 SpringSecurity에 `FilterComparator`에 등록되어 있는 Filter들을 활성화 할때 사용 가능하다.


```java
FilterComparator() {
  Step order = new Step(INITIAL_ORDER, ORDER_STEP);
  put(ChannelProcessingFilter.class, order.next());
  put(ConcurrentSessionFilter.class, order.next());
  put(WebAsyncManagerIntegrationFilter.class, order.next());
  put(SecurityContextPersistenceFilter.class, order.next());
  put(HeaderWriterFilter.class, order.next());
  put(CorsFilter.class, order.next());
  put(CsrfFilter.class, order.next());
  put(LogoutFilter.class, order.next());
  filterToOrder.put(
    "org.springframework.security.oauth2.client.web.OAuth2AuthorizationRequestRedirectFilter",
			order.next());
  filterToOrder.put(
    "org.springframework.security.saml2.provider.service.servlet.filter.Saml2WebSsoAuthenticationRequestFilter",
			order.next());
  put(X509AuthenticationFilter.class, order.next());
  put(AbstractPreAuthenticatedProcessingFilter.class, order.next());
  filterToOrder.put("org.springframework.security.cas.web.CasAuthenticationFilter",
			order.next());
  filterToOrder.put(
    "org.springframework.security.oauth2.client.web.OAuth2LoginAuthenticationFilter",
			order.next());
  filterToOrder.put(
    "org.springframework.security.saml2.provider.service.servlet.filter.Saml2WebSsoAuthenticationFilter",
			order.next());
  put(UsernamePasswordAuthenticationFilter.class, order.next());
  put(ConcurrentSessionFilter.class, order.next());
  filterToOrder.put(
    "org.springframework.security.openid.OpenIDAuthenticationFilter", order.next());
  put(DefaultLoginPageGeneratingFilter.class, order.next());
  put(DefaultLogoutPageGeneratingFilter.class, order.next());
  put(ConcurrentSessionFilter.class, order.next());
  put(DigestAuthenticationFilter.class, order.next());
  filterToOrder.put(
    "org.springframework.security.oauth2.server.resource.web.BearerTokenAuthenticationFilter", order.next());
  put(BasicAuthenticationFilter.class, order.next());
  put(RequestCacheAwareFilter.class, order.next());
  put(SecurityContextHolderAwareRequestFilter.class, order.next());
  put(JaasApiIntegrationFilter.class, order.next());
  put(RememberMeAuthenticationFilter.class, order.next());
  put(AnonymousAuthenticationFilter.class, order.next());
  filterToOrder.put(
    "org.springframework.security.oauth2.client.web.OAuth2AuthorizationCodeGrantFilter",
			order.next());
  put(SessionManagementFilter.class, order.next());
  put(ExceptionTranslationFilter.class, order.next());
  put(FilterSecurityInterceptor.class, order.next());
  put(SwitchUserFilter.class, order.next());
}
```

- `FilterComparator`에 기본적으로 등록되어 있는 Filter order 참고

```java
public HttpSecurity addFilter(Filter filter) {
  Class<? extends Filter> filterClass = filter.getClass();
  if (!comparator.isRegistered(filterClass)) {
    throw new IllegalArgumentException(
      "The Filter class "
	+ filterClass.getName()
	+ " does not have a registered order and cannot be added without a specified order. Consider using addFilterBefore or addFilterAfter instead.");
  }
  this.filters.add(filter);
  return this;
}
```
- 일반적으로 CustomFilter를 추가한다면 comparator에 등록이 되어 있지 않아 에러가 발생하게 되어 있다.
- CustomFilter를 등록할 때는 `addFilterBefore`이나 `addFilterAfter`을 사용하라고 에러 메시지가 나온다.

### addFilterBefore

```java
public HttpSecurity addFilterBefore(Filter filter,
		Class<? extends Filter> beforeFilter) {
  comparator.registerBefore(filter.getClass(), beforeFilter);
  return addFilter(filter);
}

public void registerBefore(Class<? extends Filter> filter,
		Class<? extends Filter> beforeFilter) {
  Integer position = getOrder(beforeFilter);
  if (position == null) {
    throw new IllegalArgumentException(
      "Cannot register after unregistered Filter " + beforeFilter);
  }

  put(filter, position - 1);
}

private void put(Class<? extends Filter> filter, int position) {
  String className = filter.getName();
  filterToOrder.put(className, position);
}
```
- 특정 Filter의 이전에 동작하게 Order를 설정을 할 수 있다.

```java
http.addFilterBefore(new LoggingFilter(), WebAsyncManagerIntegrationFilter.class);
```
- 해당 방식으로 등록을 하게 되면, `WebAsyncManagerIntegrationFilter`의 Order position에서 -1을 하여 LoggingFilter를 등록하게 된다.

### addFilterAfter
```java
public HttpSecurity addFilterAfter(Filter filter, Class<? extends Filter> afterFilter) {
  comparator.registerAfter(filter.getClass(), afterFilter);
  return addFilter(filter);
}

public void registerAfter(Class<? extends Filter> filter,
		Class<? extends Filter> afterFilter) {
  Integer position = getOrder(afterFilter);
  if (position == null) {
    throw new IllegalArgumentException(
      "Cannot register after unregistered Filter " + afterFilter);
  }
  put(filter, position + 1);
}
```
- 특정 Filter 이후에 동작하게 Order를 설정할 수 있다.

```java
http.addFilterAfter(new LoggingFilter(), WebAsyncManagerIntegrationFilter.class);
```
- 해당 방식으로 등록을 하게 되면, `WebAsyncManagerIntegrationFilter`의 Order position에서 +1을 하여 LoggingFilter를 등록하게 된다.
 
### addFilterAt

```java
public HttpSecurity addFilterAt(Filter filter, Class<? extends Filter> atFilter) {
  this.comparator.registerAt(filter.getClass(), atFilter);
  return addFilter(filter);
}
public void registerAt(Class<? extends Filter> filter,
		Class<? extends Filter> atFilter) {
  Integer position = getOrder(atFilter);
  if (position == null) {
    throw new IllegalArgumentException(
      "Cannot register after unregistered Filter " + atFilter);
  }

  put(filter, position);
}
```

- 특정 Filter와 같은 Order로 CustomFilter를 등록 한다.

```java
http.addFilterAt(new aLoggingFilter(), WebAsyncManagerIntegrationFilter.class);
```

- 해당 방식으로 등록을 하게 되면, `WebAsyncManagerIntegrationFilter`의 Order position과 동일하게 LoggingFilter를 등록하게 된다.

## 마치며
- Order에 대해서 설명이 없긴한데, Order는 숫자가 낮을수록 더 높은 우선순위를 가지고 있다.
- `addFilterBefore`, `addFilterAfter`를 많이 사용하는데, `addFilterAt`의 경우는 Order가 같아서 같은 Order의 Filter에서
순서를 보장하기가 어려울 수 있다.