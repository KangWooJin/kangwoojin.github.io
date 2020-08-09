---
title: "[Spring Security] Spring Security Basic - Ignoring" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-09 12:00:00+09:00
excerpt: Request 요청 시 SpringSecurity 에 걸리지 않고 무시 되는 방법에 대해서 알아 보자.
---

## 들어가며
SpringSecurity를 설정 하고 아무런 셋팅을 하지 않으면 모든 요청에 대해서 인증을 요청하는 설정이 추가가 된다.
- [SpringSecurity Configuration]({% post_url spring-security/2020-08-01-spring-security-basic-configuration %}) 참고

Web application을 개발하게 되면 js, css 등 static resource에 대해서도 요청이 들어 오게 되는데 static resource들은
인증을 확인하지 않고 무시하는게 서버 리소스를 낭비하지 않는 최고의 방법이다.

어떻게 하면 Request 요청시에 Security 설정을 타지 않고 by pass할 수 있는지에 대해서 알아 보자.

## WebSecurityConfigurerAdapter

```java
@EnableWebSecurity(debug = true)
@Order(Ordered.LOWEST_PRECEDENCE - 20)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring()
            // Spring Booot에서 제공해주는 static resource path
           .requestMatchers(PathRequest.toStaticResources().atCommonLocations())
           .antMatchers("/info");

    }
}
```

- `WebSecurityConfigurerAdapter`를 통해서 `WebSecurityConfigurer` 관련 설정을 변경할 수 있다.
- `WebSecurity`쪽에 ignoring을 통해서 Request 요청이 왔을 때 Security 설정을 무시 할 수 있게 된다.
- `ignoring` 관련해서 설정할 때 matchers 방법은 총 4가지가 존재한다.
    - mvcMatchers : URI matches 하는 방식
    - antMatchers : ant pattern을 이용해 matches하는 방식
    - regexMatchers : regex pattern을 이용해 matches하는 방식
    - requestMatchers : mvcMatcher, antMatcher, regexMatcher는 requestMatcher로 이뤄져 있어서, 완성된 Matcher를 주입할 때 사용
        
## 어떻게 ignoring 되는가?

- `ignoring`으로 등록된 경우 `SecurityFilterChain`에 Filters가 아무것도 등록이 되지 않는다.
- 즉, Security 설정을 타지 않게 된다.
- [SecurityFilterChain]({% post_url spring-security/2020-08-08-spring-security-basic-filter-chain-proxy %}) 관련 내용은 해당 포스트에서
확인할 수 있다.

### 필터가 왜 없나?

```java
@Override
protected Filter performBuild() throws Exception {
  ...
  int chainSize = ignoredRequests.size() + securityFilterChainBuilders.size();
  List<SecurityFilterChain> securityFilterChains = new ArrayList<>(
    chainSize);
  for (RequestMatcher ignoredRequest : ignoredRequests) {
    securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
  }
  for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
    securityFilterChains.add(securityFilterChainBuilder.build());
  }
  FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
  if (httpFirewall != null) {
    filterChainProxy.setFirewall(httpFirewall);
  }
  filterChainProxy.afterPropertiesSet();
  Filter result = filterChainProxy;
  
  postBuildAction.run();
  return result;
}
```

- `performBuild`시에 SecurityFilterChain을 생성하게 되는데 ignoredRequest의 경우 `DefaultSecurityFilterChain`로 생성이 되게 된다.
- 여기서 한 가지 더 주목할 부분은, ignoredRequest가 `SecurityFilterChains`에 우선순위로 등록이 되어, `FilterChainProxy`에서
먼저 매칭이 이뤄지게 되고, 매칭이 되지 않는 경우 `HttpSecurity`로 등록한 `SecurityFilterChain` 로직을 타게 된다.

```java
public DefaultSecurityFilterChain(RequestMatcher requestMatcher, Filter... filters) {
  this(requestMatcher, Arrays.asList(filters));
}
```

- `DefaultSecurityFilterChain`이 생성될 때 Filter를 파라메터로 받지만, ignoredRequest로 생성되는 경우 Filter가 빈값이 되어 `emptyList`
를 갖게 되는 것이다. 


## HttpSecurity에 permitAll과 차이
- WebSecurity에 ignoring으로 등록된 `SecurityFilterChain`의 경우는 Filter가 빈 값으로 생성되는 것을 확인하였다.
- `HttpSecurity`의 경우는 기본적으로 등록되는 Filter들이 정의가 되어 있다.

```java
protected final HttpSecurity getHttp() throws Exception {
  if (http != null) {
    return http;
  }

  AuthenticationEventPublisher eventPublisher = getAuthenticationEventPublisher();
  localConfigureAuthenticationBldr.authenticationEventPublisher(eventPublisher);

  AuthenticationManager authenticationManager = authenticationManager();
  authenticationBuilder.parentAuthenticationManager(authenticationManager);
  Map<Class<?>, Object> sharedObjects = createSharedObjects();

  http = new HttpSecurity(objectPostProcessor, authenticationBuilder,
			sharedObjects);
  if (!disableDefaults) {
    // @formatter:off
		http
			.csrf().and()
			.addFilter(new WebAsyncManagerIntegrationFilter())
			.exceptionHandling().and()
			.headers().and()
			.sessionManagement().and()
			.securityContext().and()
			.requestCache().and()
			.anonymous().and()
			.servletApi().and()
			.apply(new DefaultLoginPageConfigurer<>()).and()
			.logout();
		// @formatter:on
	ClassLoader classLoader = this.context.getClassLoader();
	List<AbstractHttpConfigurer> defaultHttpConfigurers =
			SpringFactoriesLoader.loadFactories(AbstractHttpConfigurer.class, classLoader);

	for (AbstractHttpConfigurer configurer : defaultHttpConfigurers) {
	  http.apply(configurer);
	}
  }
  configure(http);
  return http;
}
```

- `WebSecurityConfigurerAdapter`에 `configure(HttpSecurity http)`는 `WebSecurityConfigurerAdapter.getHttp`에
마지막에 있는 configure 부분을 호출해서 값을 추가하는 역할이다.
- 따라서 `HttpSecurity`에 permitAll로 ignoring한다면, 등록되어 있는 Filter를 다 확인하고 `FilterSecurityInterceptor`에서
[AccessDecisionManager]({% post_url spring-security/2020-08-08-spring-security-basic-access-decision-manager %})를 통해
Authority 를 확인 후 통과하게 된다.

## 마치며
- `WebSecurity`, `HttpSecurity` 어떻게 보면 비슷 하지만, 다른 설정을 하기 위해서 필요로 한다.
- 인증과 인가가 필요한 경우 `HttpSecurity`를 사용하게 될 것이고, ignoring과 같은 글로벌 설정을 하기 위해서 `WebSecurity`
를 사용하는게 아닐까 싶다. 
