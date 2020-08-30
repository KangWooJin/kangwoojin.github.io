---
title: "[Spring Security] Spring Security Basic - BasicAuthenticationFilter" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-23 14:00:00+09:00
excerpt: BasicAuthenticationFilter의 역할이 무엇인지 알아보자.
---

## 들어가며
SpringSecurity에서 기본적으로 제공해주는 BasicAuthenticationFilter의 역할이 무엇인지에 대해서 알아보자.

## 역할
- form 인증이 아닐 때 인증을 시도하는 필터이다.
- `Authentication` header를 이용하여 `Basic {token}` 값을 전달하여 인증을 하는 방식이다.
- token 값은 `username:password`를 BASE64로 인코딩하여 전달되는 값을 Filter에서 decode하여 인증을 진행한다.

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

    if (debug) {
      this.logger
		  .debug("Basic Authentication Authorization header found for user '" + username + "'");
    }

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
    SecurityContextHolder.clearContext();

    if (debug) {
      this.logger.debug("Authentication request for failed!", failed);
    }

    this.rememberMeServices.loginFail(request, response);

    onUnsuccessfulAuthentication(request, response, failed);

    if (this.ignoreFailure) {
      chain.doFilter(request, response);
    }
    else {
      this.authenticationEntryPoint.commence(request, response, failed);
    }

    return;
  }

  chain.doFilter(request, response);
}
``` 

- `BasicAuthenticationFilter`에 `doFilterInternal`을 통해서 인증을 진행하며, `BasicAuthenticationConverter`를 통해서
`AuthResult`인 token을 covert한다.
- `UsernamePasswordAuthenticationFilter`와의 차이가 존재하는데 Authentication을 성공 후에 처리하는 로직이 없다.
- form에서 인증을 하는 경우 이전에 페이지를 기억해두었다가 로그인 후 이전 페이지로 redirect 해주는 부분이 추가가 되어 있지만
api처럼 요청하기에 redirect를 굳이 처리할 필요가 없으니 성공 후에 처리가 비어 있다.

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

  // Fire event
  if (this.eventPublisher != null) {
    eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(
      authResult, this.getClass()));
  }

  successHandler.onAuthenticationSuccess(request, response, authResult);
}
```
- `UsernamePasswordAuthenticationFilter`에 `successfulAuthentication` 부분은 인증에 성공후에 동작하는 부분인데
successHandler를 통해서 redirect해주는 부분이 구현되어 있다.

```java
protected void onSuccessfulAuthentication(HttpServletRequest request,
	HttpServletResponse response, Authentication authResult) throws IOException {
}
```
- `BasicAuthenticationFilter`에 성공 후 처리하는 부분은 오버라이딩해서 추가적으로 개발해야 한다. 

### BasicAuthenticationConverter

```java
@Override
public UsernamePasswordAuthenticationToken convert(HttpServletRequest request) {
  String header = request.getHeader(AUTHORIZATION);
  if (header == null) {
    return null;
  }

  header = header.trim();
  if (!StringUtils.startsWithIgnoreCase(header, AUTHENTICATION_SCHEME_BASIC)) {
    return null;
  }

  if (header.equalsIgnoreCase(AUTHENTICATION_SCHEME_BASIC)) {
    throw new BadCredentialsException("Empty basic authentication token");
  }

  byte[] base64Token = header.substring(6).getBytes(StandardCharsets.UTF_8);
  byte[] decoded;
  try {
    decoded = Base64.getDecoder().decode(base64Token);
  }
  catch (IllegalArgumentException e) {
    throw new BadCredentialsException(
      "Failed to decode basic authentication token");
  }

  String token = new String(decoded, getCredentialsCharset(request));

  int delim = token.indexOf(":");

  if (delim == -1) {
    throw new BadCredentialsException("Invalid basic authentication token");
  }
  UsernamePasswordAuthenticationToken result  = new UsernamePasswordAuthenticationToken(token.substring(0, delim), token.substring(delim + 1));
  result.setDetails(this.authenticationDetailsSource.buildDetails(request));
  return result;
}
```

- Header에 `Authentication`을 가져와서 BASE64로 decode하여 username, password를 추출하게 된다.
- encode하기 전에 `username:password` 포맷으로 전달하기에 구분자 `:`을 기준으로 앞이 username, 뒤쪽이 password가 되어
`UsernamePasswordAuthenticationToken`으로 변환 후 전달 한다.

## 마치며
- `BasicAuthenticationFilter` 방식은 Session을 이용할 수 가 없어 기본 방식이 stateless 방식이 된다.
- 만약 session 처럼 인증을 매번 하고 싶지 않다면 `RememberMeServices`를 활용하여 인증정보를 캐시 하여 사용할 수 있다.
 