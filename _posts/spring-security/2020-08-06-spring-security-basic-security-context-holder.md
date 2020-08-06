---
title: "[Spring Security] Spring Security Basic - Security Context Holder" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-06 12:00:00+09:00 
excerpt: Security Context Holder가 하는 역할과 생성 과정을 알아 보자.
---

## 들어가며
Spring Security에서 사용되는 SecurityContextHolder가 하는 역할과 언제 생성되고, 
어떻게 데이터가 들어가는지 알아 보자.

## 역할

SecurityContextHolder의 역할을 정말 간단하다.

- ThreadLocal을 사용해서, request당 `Authentication` 정보 담고 있는 Holder이다.
- `Authentication`은 인증이 되어야만 생성되기에 SecurityContextHolder가 담고 있는 값을
확인하여 인증이 되었는지, 안되었는지 알 수 있다.

### SecurityContextHolder

```java
public class SecurityContextHolder {
	static {
		initialize();
	}
    ...
	private static void initialize() {
		if (!StringUtils.hasText(strategyName)) {
			// Set default
			strategyName = MODE_THREADLOCAL;
		}

		if (strategyName.equals(MODE_THREADLOCAL)) {
			strategy = new ThreadLocalSecurityContextHolderStrategy();
		}
		else if (strategyName.equals(MODE_INHERITABLETHREADLOCAL)) {
			strategy = new InheritableThreadLocalSecurityContextHolderStrategy();
		}
		else if (strategyName.equals(MODE_GLOBAL)) {
			strategy = new GlobalSecurityContextHolderStrategy();
		}
		else {
			// Try to load a custom strategy
			try {
				Class<?> clazz = Class.forName(strategyName);
				Constructor<?> customStrategy = clazz.getConstructor();
				strategy = (SecurityContextHolderStrategy) customStrategy.newInstance();
			}
			catch (Exception ex) {
				ReflectionUtils.handleReflectionException(ex);
			}
		}

		initializeCount++;
	}
    ...
}
```

- SecurityContextHolder는 static method를 통해서 서버 기동 시점에 어떤 방식의 전략으로 생성될 지
결정되어 진다.
- SpringSecurity의 default는 `MODE_THREADLOCAL`이며 Custom도 가능하다.
- 만약 값을 변경하고 싶다면, `SystemProperty`에 `spirng.security.strategy`를 변경해서 사용하면 된다.

```java
public class SecurityContextHolder {
	public static final String MODE_THREADLOCAL = "MODE_THREADLOCAL";
	public static final String MODE_INHERITABLETHREADLOCAL = "MODE_INHERITABLETHREADLOCAL";
	public static final String MODE_GLOBAL = "MODE_GLOBAL";
	public static final String SYSTEM_PROPERTY = "spring.security.strategy";
	private static String strategyName = System.getProperty(SYSTEM_PROPERTY);
    ... 
}
```

#### MODE_THREADLOCAL
- 해당 Thread에서만 사용가능한 SecurityContext를 가지고 있는 모드 이다.

#### MODE_INHERITABLETHREADLOCAL
- ThreadLocal에 있는 정보를 하위 Thread에 SecurityContext를 전파 하는 모드 이다.

#### MODE_GLOBAL         
- 해당 application에서 모든 Thread에 같은 SecurityContext를 갖게 하는 모드 이다.



### SecurityContext


```java
public interface SecurityContext extends Serializable {	
	Authentication getAuthentication();

	void setAuthentication(Authentication authentication);
}
```

- `SecurityContextHolder`가 Authentication을 갖고 있다고 했는데, 실제로는 `SecurityContext`를
갖고 있고 `SecurityContext`를 통해서 `Authentication`을 가져올 수 있다.
- `SecurityContext`의 구현체는 `SecurityContextImpl`로 `Authentication`을 갖고 있다.

```java
public class SecurityContextImpl implements SecurityContext {
    ...
	private Authentication authentication;

	public SecurityContextImpl() {}

	public SecurityContextImpl(Authentication authentication) {
		this.authentication = authentication;
	}

	@Override
	public Authentication getAuthentication() {
		return authentication;
	}

	@Override
	public void setAuthentication(Authentication authentication) {
		this.authentication = authentication;
	}
    ...
}
```

#### Authentication
 
```java
public interface Authentication extends Principal, Serializable {

	Collection<? extends GrantedAuthority> getAuthorities();

	Object getCredentials();

	Object getDetails();

	Object getPrincipal();

	boolean isAuthenticated();

	void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}
```
- `Authentication`은 사용자로부터 전달받은 데이터를 보관 및 가공하여 인증정보를 갖고 있는 인터페이스이다.
- 실제로 데이터를 갖고 있는 class는 `XXXToken`으로 끝나는 class들이다.
    - `UsernamePasswordAuthenticationToken`
    - `AnonymousAuthenticationToken`
- `GrandedAuthority`는 해당 유저의 Admin, User와 같은 권한을 체크하는데 사용 된다.
- `Credentials`는 password 정보를 담고 있는데, 인증이 완료되면 해당 데이터를 제거한다.
- `Details`는 정확하게는 모르겠지만, `remoteAddress`와 `sessionId`를 갖고 있다..
- `Principal` 인증에서 가장 중요한 데이터라 생각되는데, 유저 정보를 담고 있는 데이터 이다.
    - Object로 되어 있지만, 앞에서 얘기했던 Token class 값이다.
    - username, authorities, 유효성 체크할 수 있는 필드에 데이터를 갖고 있다.

## SecurityContextHolder 에 데이터 주입 과정
- `SecurityContextHolder`에 어떤 데이터가 들어가는지는 알아 보았으니, 언제 데이터가 담기는지
확인해 보자.

### SecurityContextPersistenceFilter
```java
public class SecurityContextPersistenceFilter extends GenericFilterBean {
    ...
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
			// Crucial removal of SecurityContextHolder contents - do this before anything
			// else.
			SecurityContextHolder.clearContext();
			repo.saveContext(contextAfterChainExecution, holder.getRequest(),
					holder.getResponse());
			request.removeAttribute(FILTER_APPLIED);

			if (debug) {
				logger.debug("SecurityContextHolder now cleared, as request processing completed");
			}
		}
	}
    ...
}
```
- `SecurityContextHolder`에 `SecurityContext`는 `SecurityContextPersistenceFilter`에서
`doFilter`가 동작될 때 생성 된다.
- session 방식을 이용 한다면, `finally` 부분에서 Session에 `SecurityContext`를 저장하고 
session에서 `SecurityContext`를 load하여 셋팅하는 부분도 포함되어 있다.
- `SecurityContext`가 생성되었다면, `SecurityContext`에 `Authenticatioon`은 언제 생성되는지 알아 보자.

### AbstractAuthenticationProcessingFilter

```java
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean
		implements ApplicationEventPublisherAware, MessageSourceAware {
	...
	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {

		...
		Authentication authResult;

		try {
			authResult = attemptAuthentication(request, response);
			if (authResult == null) {
				// return immediately as subclass has indicated that it hasn't completed
				// authentication
				return;
			}
			sessionStrategy.onAuthentication(authResult, request, response);
		}
		...

		successfulAuthentication(request, response, chain, authResult);
	}
    ...
	protected void successfulAuthentication(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain, Authentication authResult)
			throws IOException, ServletException {

		...
		SecurityContextHolder.getContext().setAuthentication(authResult);
		...
	}
    ...
}
```

- `AbstractAuthenticationProcessingFilter`는 `attemptAuthentication` 메소드 뜻 그대로 인증을 시도 하는 filter이다.
- `authResult`가 Authenticatioon 데이터를 담고 있고, `doFilter` 마지막에 성공인 경우 `successfulAuthentication`에서
`SecurityContextHolder`에 `authResult`를 저장한다.

## 마치며
- form 방식의 기본 설정만 추가하여 테스트를 진행하였기에, 다른 설정의 경우는 다르게 적용 될 수 있다
- 하지만, 동작 방식을 알았으니, 디버깅을 통해서 확인을 해볼때 어느정도 쉽게 유추가 되지 않을까 싶다!
