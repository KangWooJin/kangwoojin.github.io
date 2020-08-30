---
title: "[Spring Security] Spring Security Basic - Logout Filter" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-23 12:00:00+09:00
excerpt: Logout Filter의 역할이 무엇인지 알아보자.
---

## 들어가며
SpringSecurity에서 기본적으로 제공해주는 Logout Filter의 역할이 무엇인지에 대해서 알아보자.

## 역할
- logout을 했을 때 로그인 관련 정보를 제거해주는 역할과 logout 후 redirect 페이지로 전달하는 역할을 담당한다.

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
  HttpServletRequest request = (HttpServletRequest) req;
  HttpServletResponse response = (HttpServletResponse) res;

  if (requiresLogout(request, response)) {
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();

    if (logger.isDebugEnabled()) {
      logger.debug("Logging out user '" + auth
					+ "' and transferring to logout destination");
    }

    this.handler.logout(request, response, auth);

    logoutSuccessHandler.onLogoutSuccess(request, response, auth);

    return;
  }

  chain.doFilter(request, response);
}
```

- `requiresLogout`로 logout을 처리할 url에 매치가 된다면 logout 관련 동작을 실행 한다. 기본값은 `/logout` 이다.
- `LogoutHandler`를 통해서 logout을 처리하는 부분과 `LogoutSuccessHanlder`에서 처리하는 부분 2가지가 존재한다.

### LogoutHandler

```java
public interface LogoutHandler {
  void logout(HttpServletRequest request, HttpServletResponse response,
			Authentication authentication);
}
```

- `LogoutHandler`는 logout을 하였을 때 처리 되어야 하는 부분을 다루게 된다.
- 예를들면 csrfToken을 session에서 제거한다거나 현재 session을 만료시켜 Authentication을 못 쓰게 만드는 역할이 있다.
- default로 생성되는 `LogoutHandler`는 `CompositeLogoutHandler`로 LogoutHandler를 여러개를 처리하도록 되어 있다.
- 기본적으로 `CsrfLogoutHandler`, `SecurityContextLogoutHandler`, `LogoutSuccessEventPublishingLogoutHandler`가 등록 된다.
- `CsrfLogoutHandler`는 현재 Session에 있는 csrfToken을 만료시키는 역할을 한다.
- `SecurityContextLogoutHandler`는 현재 인증되어 있는 유저를 Session에서 만료시키는 역할을 한다.
- `LogoutSuccessEventPublishingLogoutHandler`는 logout 관련 `ApplicationEvent`을 발생 시킨다.


### LogoutSuccessHandler
- logout에 성공하였을 때 처리되는 부분을 담당한다.

```java
public class SimpleUrlLogoutSuccessHandler extends
		AbstractAuthenticationTargetUrlRequestHandler implements LogoutSuccessHandler {
  public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response,
		Authentication authentication) throws IOException, ServletException {
      super.handle(request, response, authentication);
  }
}
```
- `SimpleUrlLogoutSuccessHandler`를 default로 사용한다.
- `AbstractAuthenticationTargetUrlRequestHandler`에서 `targetUrl`로 redirect하는 역할을 담당하고 있다.

```java
private LogoutSuccessHandler createDefaultSuccessHandler() {
  SimpleUrlLogoutSuccessHandler urlLogoutHandler = new SimpleUrlLogoutSuccessHandler();
  urlLogoutHandler.setDefaultTargetUrl(logoutSuccessUrl);
  if (defaultLogoutSuccessHandlerMappings.isEmpty()) {
    return urlLogoutHandler;
  }
  DelegatingLogoutSuccessHandler successHandler = new DelegatingLogoutSuccessHandler(defaultLogoutSuccessHandlerMappings);
  successHandler.setDefaultLogoutSuccessHandler(urlLogoutHandler);
  return successHandler;
}
```

- logout configure에서 `SimpleUrlLogoutSuccessHandler`을 생성할 때 defaultTargetUrl을 셋팅하게 되는데
default 값은 `/login?logout`으로 셋팅이 된다.
- 따라서 logout을 하게 되면 다시 login 페이지로 redirect 된다.

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
  ...     
  http.logout().logoutSuccessUrl("/customLogoutSuccessUrl");
}
```
- 해당 redirect 페이지를 변경하고 싶으면 `WebSecurityConfigurerAdapter`에서 http 설정시에 `logoutSuccessUrl`를 변경하면 된다.

## 마치며
- `logout` 관련해서는 Security를 등록하면 기본적으로 등록이 되기에 변경하고 싶다면 `WebSecurityConfigurerAdapter`를 통해서
값을 오버라이딩하는 방식으로 할 수 있다.
- `logout` 후 개인정보가 남아 있지 않도록 `LogoutHandler`를 통해서 clear하는 부분을 잊지 말자~