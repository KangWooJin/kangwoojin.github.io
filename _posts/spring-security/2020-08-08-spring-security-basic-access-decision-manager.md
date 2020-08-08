---
title: "[Spring Security] Spring Security Basic - AccessDecisionManager" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-08 21:00:00+09:00
excerpt: AccessDecisionManager를 통해서 어떻게 인가(Authority)가 발생하는지 알아 보자.
---

## 들어가며
Spring Security에서 인증이 어떻게 발생하는지는 이전 포스트들에서 알아보았다.

- [UserDetailsService]({% post_url spring-security/2020-08-01-spring-security-basic-userDetailsService %})
- [SecurityContextHolder]({% post_url spring-security/2020-08-06-spring-security-basic-security-context-holder %})

인가(Authority)는 어디서 처리를 하고 있는지에 대해서 알아 본다.

## AccessDecisionManager
```java
public interface AccessDecisionManager {
	void decide(Authentication authentication, Object object,
			Collection<ConfigAttribute> configAttributes) throws AccessDeniedException,
			InsufficientAuthenticationException;

	boolean supports(ConfigAttribute attribute);

	boolean supports(Class<?> clazz);
}
```

- `AccessDeccisionManager`는 3개의 메소드를 가진 인터페이스이며, `dicide`에서 인가에 대해서 판단 한다.
- `supports`는 Voter들이 Vote에 참여할 수 있는지 없는지를 결정하는데 사용한다.
- 해당 인터페이스를 통해서 구현된 Authority 를 처리하는 class들은 3가지가 존재한다.
    - AffirmativeBased : 여러 Voter중에 하나라도 grant라면 허용 (Default)  
    - ConsensusBased : 여러 Voter들의 의견 중 grant, deny의 값이 큰걸로 판단   
    - UnanimousBased : 여러 Voter들의 의견이 전부 grant인 경우 허용
- `AccessDecisionManager`의 구현체들은 `List<AccessDecisionVoter>`를 갖고 있어, `AccessDecisionVoter`가 vote를 실시 한다.

### AffirmativeBased

```java
public class AffirmativeBased extends AbstractAccessDecisionManager {
  public AffirmativeBased(List<AccessDecisionVoter<?>> decisionVoters) {
    super(decisionVoters);
  }

	public void decide(Authentication authentication, Object object,
			Collection<ConfigAttribute> configAttributes) throws AccessDeniedException {
		int deny = 0;

		for (AccessDecisionVoter voter : getDecisionVoters()) {
			int result = voter.vote(authentication, object, configAttributes);

			if (logger.isDebugEnabled()) {
				logger.debug("Voter: " + voter + ", returned: " + result);
			}

			switch (result) {
			case AccessDecisionVoter.ACCESS_GRANTED:
				return;

			case AccessDecisionVoter.ACCESS_DENIED:
				deny++;

				break;

			default:
				break;
			}
		}

		if (deny > 0) {
			throw new AccessDeniedException(messages.getMessage(
					"AbstractAccessDecisionManager.accessDenied", "Access is denied"));
		}

		// To get this far, every AccessDecisionVoter abstained
		checkAllowIfAllAbstainDecisions();
	}
}
```

- `AffirmativeBased.devide`부분에서 `voter.vote`의 reulst가 `ACCESS_GRANTED`인 경우 error를 발생시키지 않고 넘어
가게 되어 있다.

### AccessDecisionVoter

```java
public interface AccessDecisionVoter<S> {
	int ACCESS_GRANTED = 1;
	int ACCESS_ABSTAIN = 0;
	int ACCESS_DENIED = -1;

	boolean supports(ConfigAttribute attribute);

	boolean supports(Class<?> clazz);

	int vote(Authentication authentication, S object,
			Collection<ConfigAttribute> attributes);
}
```
- `AccessDecisionVoter`는 `vote`에서 grant, abstain, denied 값을 반환 한다.
- default는 abstain으로 설정되어 있고, grant, denied, abstain 값을 전달하게 되어 있다.
- `AccessDecisionVoter`의 기본 구현체는 `WebExpressionVoter`가 생성이 된다.
- 기본 구현체는 아마 조건에 따라 다르게 생성되는데, form 방식에서는 `WebExpressionVoter`가 생성 된다.

## AccessDecisionManager은 어디서 호출되는가?

- `AccessDecisionManager`가 어디서 호출이 되어서 Authority를 판단하는지에 대해서 알아 본다.

### FilterSecurityInterceptor

```java
public class FilterSecurityInterceptor extends AbstractSecurityInterceptor implements
		Filter {
	
	public void doFilter(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {
		FilterInvocation fi = new FilterInvocation(request, response, chain);
		invoke(fi);
	}

	public void invoke(FilterInvocation fi) throws IOException, ServletException {
		if ((fi.getRequest() != null)
				&& (fi.getRequest().getAttribute(FILTER_APPLIED) != null)
				&& observeOncePerRequest) {
			// filter already applied to this request and user wants us to observe
			// once-per-request handling, so don't re-do security checking
			fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
		}
		else {
			// first time this request being called, so perform security checking
			if (fi.getRequest() != null && observeOncePerRequest) {
				fi.getRequest().setAttribute(FILTER_APPLIED, Boolean.TRUE);
			}

			InterceptorStatusToken token = super.beforeInvocation(fi);

			try {
				fi.getChain().doFilter(fi.getRequest(), fi.getResponse());
			}
			finally {
				super.finallyInvocation(token);
			}

			super.afterInvocation(token, null);
		}
	}

}
```

- `FilterChainProxy`에서 `SecurityFilterChain`을 가져오고, `SecurityFilterChain`의 많은 Filters 중에
마지막에 `FilterSecurityInterceptor`가 존재한다.
- 마지막 Filter답게, `chain.doFilter`를 호출하는 부분이 없다.
- `FilterSecurityInterceptor`를 통해서 Authority를 검증하게 되는데, 실제 로직은 `AbstractSecurityInterceptor`에서 구현되어 있다.

### AbstractSecurityInterceptor

```java
public abstract class AbstractSecurityInterceptor implements InitializingBean,
		ApplicationEventPublisherAware, MessageSourceAware {
	
	protected InterceptorStatusToken beforeInvocation(Object object) {
		...
		// Attempt authorization
		try {
			this.accessDecisionManager.decide(authenticated, object, attributes);
		}
		catch (AccessDeniedException accessDeniedException) {
			publishEvent(new AuthorizationFailureEvent(object, attributes, authenticated,
					accessDeniedException));

			throw accessDeniedException;
		}
        ...
	}
}
```

- `AbstractSecurityInterceptor`에서 `beforeInvocation`을 할 때 `accessDecisionManager.decide`을 호출하여
Voter들에게 vote를 하고, 에러가 발생하는지 안하는지로 Authority를 판단하게 되어 있다.
- `AccessDeniedException`가 발생한 경우 `AccessDeniedHandler`에서 받아서 에러 핸들링을 이어 갈 수 있다.

## 마치며
- form 방식과 token 방식이 서로 다르게 구현되어 있는데, form 방식에 대해서 알아 보았다.
- 정말 간단하게 어떻게 호출이 발생되는지를 알아 보았는데 실제로 Voter들이 어떻게 Authority를 판단하는 부분까지는 다음에
기회가 된다면 작성할 생각이다..