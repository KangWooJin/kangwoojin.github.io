---
title: "[Spring Security] Spring Security Basic - CSRF Filter" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-23 12:00:00+09:00
excerpt: CSRF Filter의 역할이 무엇인지 알아보고 어떻게 동작 하는지 알아보자.
---

## 들어가며
SpringSecurity를 설정하면 자동으로 설정되는 Filter에 대해서 정리하고 있다.

그 중에서 자동으로 등록되는 CSRF Filter에 대해서 알아보자.

## CSRF(Cross Site Request Forgery) 란?
- 인증된 유저의 계정을 사용해 악의적인 변경 요청을 만들어 보내는 기법
- [https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF))
- [https://namu.wiki/w/CSRF](https://namu.wiki/w/CSRF)

## 역할

![csrf-attack](/assets/images/spring-security/csrf-attack.png)

- 사용자가 은행에 로그인을 하고 해당 브라우저에 유효한 쿠키를 가지고 있는 상태를 충족한다.
- 사용자가 아무런 사이트를 돌아다니다가, 공격자 웹 사이트에 접근을 하니, 공격자 웹 사이트에 숨겨져 있는 스크립트가 동작하면서
사용자의 유효한 쿠키를 바탕으로 스크립트가 동작한다.
- 계좌이체를 실행하는 스크립트가 동작하고, 유효한 쿠키를 바탕으로 은행에서는 스크립트를 사용자로 판단하여 계좌이체를 실행한다.
- 이러한 과정이 발생되는게 CSRF Attack인데, 이러한 공격을 막기 위해 CSRF Filter를 이용할 수 있다. 
- CSRF Filter의 역할을 resource를 변경하는 요청이 들어왔을 때 정상적인 경로로 요청이 들어왔는지 확인하는 역할을 담당 한다.  

## CsrfFilter 동작 방식

```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {
  request.setAttribute(HttpServletResponse.class.getName(), response);


  // 1. 저장되어 있는 csrf token 확인
  CsrfToken csrfToken = this.tokenRepository.loadToken(request);
  final boolean missingToken = csrfToken == null;
  if (missingToken) {
    // 2. csrf token이 없는 경우 서버에서 csrf token 생성
    csrfToken = this.tokenRepository.generateToken(request);
    this.tokenRepository.saveToken(csrfToken, request, response);
  }
  request.setAttribute(CsrfToken.class.getName(), csrfToken);
  request.setAttribute(csrfToken.getParameterName(), csrfToken);

  // 3. csrf filter 사용 유무 확인
  if (!this.requireCsrfProtectionMatcher.matches(request)) {
    filterChain.doFilter(request, response);
    return;
  }
 
  // 4. request로 부터 csrf token 추출
  String actualToken = request.getHeader(csrfToken.getHeaderName());
  if (actualToken == null) {
    actualToken = request.getParameter(csrfToken.getParameterName());
  }
  // 5. 서버에서 생성한 csrf token과 request로 받은 csrf token 비교
  if (!csrfToken.getToken().equals(actualToken)) {
    if (this.logger.isDebugEnabled()) {
      this.logger.debug("Invalid CSRF token found for "
					+ UrlUtils.buildFullRequestUrl(request));
    }
    if (missingToken) {
      this.accessDeniedHandler.handle(request, response,
					new MissingCsrfTokenException(actualToken));
    }
    else {
      this.accessDeniedHandler.handle(request, response,
					new InvalidCsrfTokenException(csrfToken, actualToken));
    }
    return;
  }

  filterChain.doFilter(request, response);
}
``` 

- CsrfFilter에 동작 부분의 일부를 확인해보자.
- SpringSecurity에서 제공하는 CsrfFilter 동작 방식은 csrf Token을 이용해 인증하는 방식을 사용한다.

1. 저장되어 있는 csrf token 확인
2. csrf token이 없는 경우 서버에서 csrf token 생성
3. csrf filter 사용 유무 확인
4. request로 부터 csrf token 추출
5. 서버에서 생성한 csrf token과 request로 받은 csrf token 비교
6. csrf token 불 일치시에 `AccessDeniedException` 발생


## 마치며
- csrf token을 이용해서 유효한 요청인지 확인을 할 수 있지만, form을 이용하지 않고 Json api를 이용하는 방식에서는
csrf token을 제공해주고 그 값을 가지고 요청하기가 어려울 수 있다.
- 그래서 Referrer 검증을 통해서 csrf token 대신 해결하는 방법도 존재한다.