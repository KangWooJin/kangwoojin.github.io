---
title: "[Spring Security] Spring Security Basic - HeaderWriterFilter" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-22 12:00:00+09:00
excerpt: HeaderWriterFilter가 언제 생성되고 어떤 역할을 하고 있는지 알아 보자.
---

## 들어가며
SpringSecurity를 추가하게 되면 Spring 프레임워크에서 기본적으로 Security 관련 Header를 추가해 준다.

Security관련 Header를 추가해주는 Filter가 `HeaderWriterFilter`인데, 
`HeaderWriterFilter`가 언제 생성되고, 어떤 `HeaderWriter`가 생성는지, 그리고 각 `HeaderWriter`이 하는 역할이 무엇인지 알아보자.

## 생성 과정

```java
@Override
public void configure(H http) {
    HeaderWriterFilter headersFilter = createHeaderWriterFilter();
    http.addFilter(headersFilter);
}
```

- `HeadersConfigurer`에 configure 부분에서 `HeaderWriterFilter`를 생성한다. 
- configure의 발생 시점은 `springSecurityFilterChain`이 `bean`으로 등록될 때 
build 하면서 configure를 하는 과정 중 하나 이다.

```java
private HeaderWriterFilter createHeaderWriterFilter() {
  List<HeaderWriter> writers = getHeaderWriters();
  if (writers.isEmpty()) {
    throw new IllegalStateException(
      "Headers security is enabled, but no headers will be added. Either add headers or disable headers security");
  }
  HeaderWriterFilter headersFilter = new HeaderWriterFilter(writers);
  headersFilter = postProcess(headersFilter);
  return headersFilter;
}
```

- `HeaderWriterFilter`는 여러 `HeaderWriter`를 갖고 있다가, security 관련 header를 추가해주는 역할을 하게 된다.

```java
private List<HeaderWriter> getHeaderWriters() {
  List<HeaderWriter> writers = new ArrayList<>();
  addIfNotNull(writers, contentTypeOptions.writer);
  addIfNotNull(writers, xssProtection.writer);
  addIfNotNull(writers, cacheControl.writer);
  addIfNotNull(writers, hsts.writer);
  addIfNotNull(writers, frameOptions.writer);
  addIfNotNull(writers, hpkp.writer);
  addIfNotNull(writers, contentSecurityPolicy.writer);
  addIfNotNull(writers, referrerPolicy.writer);
  addIfNotNull(writers, featurePolicy.writer);
  writers.addAll(headerWriters);
  return writers;
}
```

- 기본적으로 Security에서 제공해주는 `HeaderWriters`는 9개이지만, `enabled`가 default인 것은 위에서 5개만 되어 있다.
- 따라서 기본적으로 등록되는 `HeaderWriter`는 총 5개 이다.

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
  http.headers().referrerPolicy().and().addHeaderWriter(xxx);
}
```

- custom header를 추가적으로 등록하거나 enabled가 아닌 header를 등록하기 위해서는, 
`WebSecurityConfigurerAdapter`에서 http configure를 오버라이딩하여 `http.headers()`를 통해서 추가 할 수 있다.



```java
public class HeaderWriterFilter extends OncePerRequestFilter {

	@Override
	protected void doFilterInternal(HttpServletRequest request,
			HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		if (this.shouldWriteHeadersEagerly) {
			doHeadersBefore(request, response, filterChain);
		} else {
			doHeadersAfter(request, response, filterChain);
		}
	}

	private void doHeadersAfter(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws IOException, ServletException {
		HeaderWriterResponse headerWriterResponse = new HeaderWriterResponse(request,
				response);
		HeaderWriterRequest headerWriterRequest = new HeaderWriterRequest(request,
				headerWriterResponse);
		try {
			filterChain.doFilter(headerWriterRequest, headerWriterResponse);
		} finally {
			headerWriterResponse.writeHeaders();
		}
	}

	void writeHeaders(HttpServletRequest request, HttpServletResponse response) {
		for (HeaderWriter writer : this.headerWriters) {
			writer.writeHeaders(request, response);
		}
	}
}
```
 
- `HeaderWriterFilter`에서는 `shouldWriteHeadersEagerly` 설정에 따라서 response에 Header를 요청 전, 요청 후에
header를 추가할지를 판단한다. 기본값은 false라서 요청 후에 추가하게 되어 있다.
- filter가 종료 되고 finally 부분에 response에 security 관련 header를 추가하게 된다. 

## HeaderWriter 역할

### XContentTypeOptionsHeaderWriter

- 마임(MIME) 타입 스니핑 방어
- [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options)
- header에 해당 값을 추가 하면 방어할 수 있다.
 
```
X-Content-Type-Options: nosniff
```


### XXssProtectionHeaderWriter

- 브라우저에 내장된 XSS 필터 적용
- [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-XSS-Protection](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-XSS-Protection)
- [https://github.com/naver/lucy-xss-filter](https://github.com/naver/lucy-xss-filter)

```
X-XSS-protection: 1; mode=block
```

### CacheControlHeaderWriter

- 캐시 히스토리 취약점 방어
- [https://www.owasp.org/index.php/Testing_for_Browser_cache_weakness_(OTG-AUTHN-006)](https://www.owasp.org/index.php/Testing_for_Browser_cache_weakness_(OTG-AUTHN-006))

```
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Expires: 0
Pragma: no-cache
```

### HstsHeaderWriter

- HTTP로 페이지에 접근하더라도 자동으로 HTTPS 페이지로 접근하게 해준다.
- [https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Strict_Transport_Security_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Strict_Transport_Security_Cheat_Sheet.html)

```
Strict-Transport-Security: max-age=31536000 ; includeSubDomains
```

### XFrameOptionsHeaderWriter

- clickjacking 방어
- [https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options)

```
X-Frame-Options: DENY
```


## 마치며
- 앞에서 설명했던 Header에 대해서 한국어로 설명이 되어 있는 블로그를 참고해도 좋을 것 같다!
- [https://cyberx.tistory.com/171](https://cyberx.tistory.com/171)  