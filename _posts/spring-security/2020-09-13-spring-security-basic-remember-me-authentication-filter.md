---
title: "[Spring Security] Spring Security Basic - RememberMeAuthenticationFilter" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-09-19 14:00:00+09:00
excerpt: RememberMeAuthenticationFilter 어떤 역할을 하는지 알아보자.
---

## 들어가며
RememberMeAuthenticationFilter 어떤 역할을 하는지 알아보자.

## 역할
- 세션이 사라지거나 만료가 되더라도 쿠키 또는 DB를 사용하여 저장된 토큰 기반으로 인증을 지원하는 필터이다.

## 필터 등록 과정

```java
http.rememberMe()
    .userDetailsService(userDetailsService);
```

- `WebSecurityConfigurerAdapter`에서 http를 이용해서 rememberMe filter를 등록해야 한다.
- `UserDetailsService`를 등록하지 않으면 에러가 발생하기에, `UserDetailsService`를 명시적으로 등록해줘야 한다.
- `rememberMe()`를 하게 되면, `RememberMeConfigurer`에서 초기화가 발생하게 된다.

```java
public void init(H http) throws Exception {
  validateInput();
  String key = getKey();
  RememberMeServices rememberMeServices = getRememberMeServices(http, key);
  http.setSharedObject(RememberMeServices.class, rememberMeServices);
  LogoutConfigurer<H> logoutConfigurer = http.getConfigurer(LogoutConfigurer.class);
  if (logoutConfigurer != null && this.logoutHandler != null) {
    logoutConfigurer.addLogoutHandler(this.logoutHandler);
  }

  RememberMeAuthenticationProvider authenticationProvider = new RememberMeAuthenticationProvider(
    key);
  authenticationProvider = postProcess(authenticationProvider);
  http.authenticationProvider(authenticationProvider);

  initDefaultLoginFilter(http);
}
```
- `RememberMeConfigurer`에서 초기화 시에 `RememberMeServices`를 등록하게 되는데, 기본 값으로 `TokenBasedRememberMeServices`
가 등록이 된다.
- SpringSecurity에서 제공하는 `RememberMeServices` 구현체는 `TokenBasedRememberMeServices`와 `PersistentTokenBasedRememberMeServices`가 존재한다.
- 둘다 cookie를 이용하지만, `PersistentTokenBasedRememberMeServices`는 `TokenRepository`를 이용해서 저장소로부터 token이 존재하는지 확인하여 인증을 진행한다.
- `TokenBasedRememberMeServices`는 `expectedTokenSignature`라는 값을 생성하여, cookie로부터 전달 받은 데이터가 해당 값과 동일한지 체크하는 부분으로 인증을 진행 한다.
 
```java
@Override
public void configure(H http) {
  RememberMeAuthenticationFilter rememberMeFilter = new RememberMeAuthenticationFilter(
    http.getSharedObject(AuthenticationManager.class),
			this.rememberMeServices);
  if (this.authenticationSuccessHandler != null) {
    rememberMeFilter
				.setAuthenticationSuccessHandler(this.authenticationSuccessHandler);
  }
  rememberMeFilter = postProcess(rememberMeFilter);
  http.addFilter(rememberMeFilter);
}
```
- 앞에서 초기화를 진행해두었으니, 이제 실제로 configure 단계에서 필터를 등록하게 된다.
- RememberMeAuthenticationFilter는 기본으로 등록되는 Filter가 아니기에 필요시에 추가해서 사용하면 된다.
 
## 토큰(cookie) 생성 과정

```java
public interface RememberMeServices {
  Authentication autoLogin(HttpServletRequest request, HttpServletResponse response);
	
  void loginFail(HttpServletRequest request, HttpServletResponse response);

  void loginSuccess(HttpServletRequest request, HttpServletResponse response,
		Authentication successfulAuthentication);
}
```

- `RememberMeServices`가 하는 역할은 3가지를 담당한다.
- `autoLogin`은 cookie를 확인하여 해당 cookie로 부터 user 인증을 진행한다.
- `loginFail`은 인증 실패하였을 때 해당 cookie의 값을 빈 값으로 전달하게 한다. 추가 동작 부분은 비어 있어서 추가적으로 구현을 해야 한다.
- `loginSuccess`에서 인증에 성공하였으니 response에 cookie을 할당하게 된다.

### autoLogin
```java
@Override
public final Authentication autoLogin(HttpServletRequest request,
		HttpServletResponse response) {
  String rememberMeCookie = extractRememberMeCookie(request);

  if (rememberMeCookie == null) {
    return null;
  }

  logger.debug("Remember-me cookie detected");

  if (rememberMeCookie.length() == 0) {
    logger.debug("Cookie was empty");
    cancelCookie(request, response);
    return null;
  }

  UserDetails user = null;

  try {
    String[] cookieTokens = decodeCookie(rememberMeCookie);
    user = processAutoLoginCookie(cookieTokens, request, response);
    userDetailsChecker.check(user);

    logger.debug("Remember-me cookie accepted");

    return createSuccessfulAuthentication(request, user);
  }
  catch (CookieTheftException cte) {
    cancelCookie(request, response);
    throw cte;
  }
  catch (UsernameNotFoundException noUser) {
    logger.debug("Remember-me login was valid but corresponding user not found.",
				noUser);
  }
  catch (InvalidCookieException invalidCookie) {
    logger.debug("Invalid remember-me cookie: " + invalidCookie.getMessage());
  }
  catch (AccountStatusException statusInvalid) {
    logger.debug("Invalid UserDetails: " + statusInvalid.getMessage());
  }
  catch (RememberMeAuthenticationException e) {
    logger.debug(e.getMessage());
  }

  cancelCookie(request, response);
  return null;
}
```
- `TokenBasedRememberMeServices`는 `AbstractRememberMeServices`를 상속 받고 있어 일부 메소드만 추가 구현이 되어 있다.
- cookie를 추출하고, 확인하는 부분은 `AbstractRememberMeService`에서 진행하고, cookie을 바탕으로 user를 가져오는 부분인 `processAutoLoginCookie` 부분은
 `TokenBasedRememberMeServices`에서 진행하게 된다.
 
```java
@Override
protected UserDetails processAutoLoginCookie(String[] cookieTokens,
		HttpServletRequest request, HttpServletResponse response) {

  if (cookieTokens.length != 3) {
    throw new InvalidCookieException("Cookie token did not contain 3"
				+ " tokens, but contained '" + Arrays.asList(cookieTokens) + "'");
  }

  long tokenExpiryTime;

  try {
    tokenExpiryTime = new Long(cookieTokens[1]);
  }
  catch (NumberFormatException nfe) {
    throw new InvalidCookieException(
      "Cookie token[1] did not contain a valid number (contained '"
					+ cookieTokens[1] + "')");
  }

  if (isTokenExpired(tokenExpiryTime)) {
    throw new InvalidCookieException("Cookie token[1] has expired (expired on '"
				+ new Date(tokenExpiryTime) + "'; current time is '" + new Date()
				+ "')");
  }

  
  UserDetails userDetails = getUserDetailsService().loadUserByUsername(
    cookieTokens[0]);

  Assert.notNull(userDetails, () -> "UserDetailsService " + getUserDetailsService()
			+ " returned null for username " + cookieTokens[0] + ". "
			+ "This is an interface contract violation");

  
  String expectedTokenSignature = makeTokenSignature(tokenExpiryTime,
			userDetails.getUsername(), userDetails.getPassword());

  if (!equals(expectedTokenSignature, cookieTokens[2])) {
    throw new InvalidCookieException("Cookie token[2] contained signature '"
				+ cookieTokens[2] + "' but expected '" + expectedTokenSignature + "'");
  }

  return userDetails;
}
``` 

- cookie의 만료를 확인하고 cookie에 username을 추출하여 `UserDetailsService`에서 UserDetails인 AuthenticationToken을
가져오게 된다.

### loginFail
```java
@Override
public final void loginFail(HttpServletRequest request, HttpServletResponse response) {
  logger.debug("Interactive login attempt was unsuccessful.");
  cancelCookie(request, response);
  onLoginFail(request, response);
}

protected void onLoginFail(HttpServletRequest request, HttpServletResponse response) {
}
```
- `loginFail`에는 cookie를 새롭게 만들어서 전달하게 된다.

### loginSuccess
```java
@Override
public final void loginSuccess(HttpServletRequest request,
		HttpServletResponse response, Authentication successfulAuthentication) {

  if (!rememberMeRequested(request, parameter)) {
    logger.debug("Remember-me login not requested.");
    return;
  }

  onLoginSuccess(request, response, successfulAuthentication);
}
```
- `loginSuccess` 부분은 인증에 성공하는 filter에서 호출이 발생하게 된다.
- 단 request에 `remeber-me`와 같이 remeberMe 기능을 사용할지에 대해서 request parameter로 전달되어야 한다. 기본값이 `remember-me`이다.


```java
protected void successfulAuthentication(HttpServletRequest request,
	HttpServletResponse response, FilterChain chain, Authentication authResult)
	throws IOException, ServletException {

  if (logger.isDebugEnabled()) {
    logger.debug("Authentication success. Updating SecurityContextHolder to contain: "
				+ authResult);
  }

  SecurityContextHolder.getContext().setAuthentication(authResult);

  rememberMeServices.loginSuccess(request, response, authResult);

  if (this.eventPublisher != null) {
    eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(
    authResult, this.getClass()));
  }

  successHandler.onAuthenticationSuccess(request, response, authResult);
}
```

- `UsernamePasswordAuthenticationFilter`에 abstract class인 `AbstractAuthenticationProcessingFilter`에서 `successfulAuthentication`할 때 rememberMeServices에
loginSuccess를 호출하게 된다.

```java
@Override
protected void doFilterInternal(HttpServletRequest request,
		HttpServletResponse response, FilterChain chain)
				throws IOException, ServletException {
  final boolean debug = this.logger.isDebugEnabled();
  try {
    UsernamePasswordAuthenticationToken authRequest = authenticationConverter.convert(request);
    if (authRequest == null) {
      chain.doFilter(request, response);
      return;
    }
    String username = authRequest.getName();
    
    if (authenticationIsRequired(username)) {
      Authentication authResult = this.authenticationManager
					.authenticate(authRequest);

      if (debug) {
        this.logger.debug("Authentication success: " + authResult);
      }

      SecurityContextHolder.getContext().setAuthentication(authResult);

      this.rememberMeServices.loginSuccess(request, response, authResult);

      onSuccessfulAuthentication(request, response, authResult);
    }

  }
  catch (AuthenticationException failed) {
		...
	chain.doFilter(request, response);
  }
}
```
- `BasicAuthenticationFilter`에서도 remeberMeServices에 loginSuccess를 호출하게 된다.
     
## 인증 과정

![login-page](/assets/images/spring-security/login-page.png)

1. login시에 remember me on this computer를 check하게 되면 request parameter로 remeber-me에 값이 서버로 전달되게 된다.
2. 인증을 담당하는 AuthenticationFilter에서 success를 하게 되면 remeberMe 관련 parameter를 확인 후 cookie를 생성하게 된다.
3. 추후에 유저가 다시 브라우저에 접근을 하였을 때 cookie가 남아 있고 인증이 되어 있지 않다면, remeberMe cookie를 이용해서 autoLogin이 되게 된다.


## 마치며
- Session 방식의 인증을 사용하지 않고도, 유저가 여러번 로그인을 하지 않고 인증을 유지할 수 있는 방법에 대해서 공부하였다.
- 많은 웹사이트들이 해당 기능을 지원해주고 있는데, 다들 SpringSecurity를 사용하고 있는지 궁금해졌다ㅎㅎ 