---
title: "[Spring Security] Spring Security Basic - Filter Chain Proxy" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-08 12:00:00+09:00
last_modified_at: 2020-08-09 15:00:00  
excerpt: FilterChainProxy가 언제 생성 및 호출 되고 무엇을 담당하고 있는지 알아 보자.
---

## 들어가며
Spring Security는 여러개의 Filter를 사용하여, 인증 및 인가를 진행 한다. 

해당 필터들은 chain형태로 연속적으로 진행되는데 그때 사용 되는 class가 `FilterChainProxy`이다.

`FilterChainProxy`가 언제 생성 및 호출 되고 무엇을 담당하고 있는지 알아 보자. 

## FilterChainProxy 생성 과정

### WebSecurityConfiguration


```java
@Configuration(proxyBeanMethods = false)
public class WebSecurityConfiguration implements ImportAware, BeanClassLoaderAware {
	...
	@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public Filter springSecurityFilterChain() throws Exception {
		boolean hasConfigurers = webSecurityConfigurers != null
				&& !webSecurityConfigurers.isEmpty();
		if (!hasConfigurers) {
			WebSecurityConfigurerAdapter adapter = objectObjectPostProcessor
					.postProcess(new WebSecurityConfigurerAdapter() {
					});
			webSecurity.apply(adapter);
		}
		return webSecurity.build();
	}
}
``` 

- `FilterChainProxy`는 `springSecurityFilterChain` bean이 등록될 때 생성 된다.

```java
@Override
protected final O doBuild() throws Exception {
  synchronized (configurers) {
    buildState = BuildState.INITIALIZING;

    beforeInit();
    init();
    buildState = BuildState.CONFIGURING;

    beforeConfigure();
    configure();
    buildState = BuildState.BUILDING;

    O result = performBuild();
    buildState = BuildState.BUILT;

    return result;
  }
}
```

- `webSecurity.build()`가 `AbstractConfiguredSecurityBuilder.doBuild`로 넘어가게 되는데,
이때 init, configure, performBuild가 진행 된다.
- 우리가 원하는 `FilterChainProxy`의 생성은 `performBuild`시에 이뤄진다.

```java
@Override
protected Filter performBuild() throws Exception {
  Assert.state(!securityFilterChainBuilders.isEmpty(),
				() -> "At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. "
						+ "Typically this done by adding a @Configuration that extends WebSecurityConfigurerAdapter. "
						+ "More advanced users can invoke "
						+ WebSecurity.class.getSimpleName()
						+ ".addSecurityFilterChainBuilder directly");
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
  if (debugEnabled) {
			logger.warn("\n\n"
					+ "********************************************************************\n"
					+ "**********        Security debugging is enabled.       *************\n"
					+ "**********    This may include sensitive information.  *************\n"
					+ "**********      Do not use in a production system!     *************\n"
					+ "********************************************************************\n\n");
			result = new DebugFilter(filterChainProxy);
  }
  postBuildAction.run();
  return result;
}
```

- `FilterChainProxy`에 `SecurityFilterChain`도 여기서 생성이 된다.
- debugMode를 활성화 한 경우는 `DebugFilter`로 `FilterChainProxy`를 한번 더 감싸서 `DelegatingFilterProxy`에서
 호출시 `DebugFilter` -> `FilterChainProxy` 순으로 호출이 된다.

## FilterChainProxy 호출 과정

### SecurityFilterAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(SecurityProperties.class)
@ConditionalOnClass({ AbstractSecurityWebApplicationInitializer.class, SessionCreationPolicy.class })
@AutoConfigureAfter(SecurityAutoConfiguration.class)
public class SecurityFilterAutoConfiguration {
	private static final String DEFAULT_FILTER_NAME = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME;

	@Bean
	@ConditionalOnBean(name = DEFAULT_FILTER_NAME)
	public DelegatingFilterProxyRegistrationBean securityFilterChainRegistration(
			SecurityProperties securityProperties) {
		DelegatingFilterProxyRegistrationBean registration = new DelegatingFilterProxyRegistrationBean(
				DEFAULT_FILTER_NAME);
		registration.setOrder(securityProperties.getFilter().getOrder());
		registration.setDispatcherTypes(getDispatcherTypes(securityProperties));
		return registration;
	}
    ...
}
```

- `SecurityFilterAutoConfiguration`에서 `DelegatingFilterProxyRegistrationBean`을 bean으로 등록하는데 그때 
`DEFAULT_FILTER_NAME`은 `springSecurityFilterChain`으로 전달 한다.

### DelegatingFilterProxyRegistrationBean

```java
public class DelegatingFilterProxyRegistrationBean extends AbstractFilterRegistrationBean<DelegatingFilterProxy>
		implements ApplicationContextAware {
    ...
	@Override
	public DelegatingFilterProxy getFilter() {
		return new DelegatingFilterProxy(this.targetBeanName, getWebApplicationContext()) {

			@Override
			protected void initFilterBean() throws ServletException {
				// Don't initialize filter bean on init()
			}

		};
	}
}
```

- `DelegatingFilterProxyRegistrationBean`에 `getFilter`가 호출될 때 `DelegatingFilterProxy`를 생성하게 된다.

### DelegatingFilterProxy

```java
public class DelegatingFilterProxy extends GenericFilterBean {
    ...
	@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		// Lazily initialize the delegate if necessary.
		Filter delegateToUse = this.delegate;
		if (delegateToUse == null) {
			synchronized (this.delegateMonitor) {
				delegateToUse = this.delegate;
				if (delegateToUse == null) {
					WebApplicationContext wac = findWebApplicationContext();
					if (wac == null) {
						throw new IllegalStateException("No WebApplicationContext found: " +
								"no ContextLoaderListener or DispatcherServlet registered?");
					}
					delegateToUse = initDelegate(wac);
				}
				this.delegate = delegateToUse;
			}
		}

		// Let the delegate perform the actual doFilter operation.
		invokeDelegate(delegateToUse, request, response, filterChain);
	}
    ...
	protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
		String targetBeanName = getTargetBeanName();
		Assert.state(targetBeanName != null, "No target bean name set");
		Filter delegate = wac.getBean(targetBeanName, Filter.class);
		if (isTargetFilterLifecycle()) {
			delegate.init(getFilterConfig());
		}
		return delegate;
	}
}
```

- `DelegatingFilterProxy`는 첫 요청이 들어올 때까지 `delegateToUse`를 초기화 하지 않고 첫 요청 시에 Lazy 초기화를 이용 한다.
- `initDelegate`시에 `springSecurityFilterChain`를 delegate로 등록하게 되는데, delegate로 등록된 Filter가 `FilterChainProxy`이다.
- debugMode를 활성화 한경우는 `DebugFilter`가 `FilterChainProxy`를 한번 더 감싼 형태로 되어 있다.
- 모든 요청은 `DelegatingFilterProxy` -> `FilterChainProxy` 순으로 들어 와서 `FilterChainProxy`에 있는
`SecurityFilterChain`을 확인하여 동작하게 된다.


### AbstractSecurityWebApplicationInitializer
- 만약 SpringBoot를 사용하지 않는 다면, `AbstractSecurityWebApplicationInitializer`를 이용하여 `DelegatingFilterProxy`
를 생성할 수 있다.

```java
public abstract class AbstractSecurityWebApplicationInitializer
		implements WebApplicationInitializer {

	public final void onStartup(ServletContext servletContext) {
		beforeSpringSecurityFilterChain(servletContext);
		if (this.configurationClasses != null) {
			AnnotationConfigWebApplicationContext rootAppContext = new AnnotationConfigWebApplicationContext();
			rootAppContext.register(this.configurationClasses);
			servletContext.addListener(new ContextLoaderListener(rootAppContext));
		}
		if (enableHttpSessionEventPublisher()) {
			servletContext.addListener(
					"org.springframework.security.web.session.HttpSessionEventPublisher");
		}
		servletContext.setSessionTrackingModes(getSessionTrackingModes());
		insertSpringSecurityFilterChain(servletContext); // 여기
		afterSpringSecurityFilterChain(servletContext);
	}

	private void insertSpringSecurityFilterChain(ServletContext servletContext) {
		String filterName = DEFAULT_FILTER_NAME;
		DelegatingFilterProxy springSecurityFilterChain = new DelegatingFilterProxy(
				filterName);
		String contextAttribute = getWebApplicationContextAttribute();
		if (contextAttribute != null) {
			springSecurityFilterChain.setContextAttribute(contextAttribute);
		}
		registerFilter(servletContext, true, filterName, springSecurityFilterChain);
	}
}
```

## FilterChainProxy 역할

- `WebSecurityConfigurerAdapter`를 bean으로 등록하면, security RequestPath 마다 어떻게 처리할지를 결정하게 되는데
RequestPath를 확인하여 어떤 `WebSecurityConfigurerAdapter`를 이용하게 할지를 결정 한다.
- 즉 Request마다 어떤 `SecurityFilterChain`을 이용하게 될지를 선택하는 역할!


```java
public class FilterChainProxy extends GenericFilterBean {
	...
	@Override
	public void doFilter(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {
		boolean clearContext = request.getAttribute(FILTER_APPLIED) == null;
		if (clearContext) {
			try {
				request.setAttribute(FILTER_APPLIED, Boolean.TRUE);
				doFilterInternal(request, response, chain);
			}
			finally {
				SecurityContextHolder.clearContext();
				request.removeAttribute(FILTER_APPLIED);
			}
		}
		else {
			doFilterInternal(request, response, chain);
		}
	}

	private void doFilterInternal(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {

		FirewalledRequest fwRequest = firewall
				.getFirewalledRequest((HttpServletRequest) request);
		HttpServletResponse fwResponse = firewall
				.getFirewalledResponse((HttpServletResponse) response);

		List<Filter> filters = getFilters(fwRequest);

		if (filters == null || filters.size() == 0) {
			if (logger.isDebugEnabled()) {
				logger.debug(UrlUtils.buildRequestUrl(fwRequest)
						+ (filters == null ? " has no matching filters"
								: " has an empty filter list"));
			}

			fwRequest.reset();

			chain.doFilter(fwRequest, fwResponse);

			return;
		}

		VirtualFilterChain vfc = new VirtualFilterChain(fwRequest, chain, filters);
		vfc.doFilter(fwRequest, fwResponse);
	}

    private List<Filter> getFilters(HttpServletRequest request) {
	  for (SecurityFilterChain chain : filterChains) {
	    if (chain.matches(request)) {
	      return chain.getFilters();
	    }
	  }
	  return null;
	}

}
```

- Filter의 시작은 `doFilter`를 통해서 시작되며, `doFilterInternal`에서 `getFilters`를 통해서 Request에 맞는 `SecurityFilterChain`
에 Filters을 가져 온다.
- `chain.matches`는 HttpSecurity, WebSecurity에서 설정한 `mvcMatchers`, `antMatchers`, `regexMatkchers`, `requestMatchers`에
해당 된다면 해당 HttpSecurity, WebSecurity에 설정된 Filter를 사용하게 된다.

```java
@Override
public void configure(WebSecurity web) throws Exception {
  web.ignoring()
     .requestMatchers(PathRequest.toStaticResources().atCommonLocations())
     .antMatchers("/info");
}

@Override
protected void configure(HttpSecurity http) throws Exception {
  http.authorizeRequests()
      .mvcMatchers("/", "/info", "/account/**").permitAll()
      .mvcMatchers("/admin").hasAnyRole("ADMIN")
      .mvcMatchers("/user").hasAnyRole("USER")
      .anyRequest().authenticated()
      .accessDecisionManager(accessDecisionManager());
  http.formLogin();
  http.httpBasic();
}
```

- 만약 `WebSecurityConfigurerAdapter`의 configure를 이렇게 셋팅 한경우 `SecurityFilterChain`은 총 3개가 등록이 된다.
- `WebSecurity`에서 requestMatchers, antMatchers로 2개, `HttpSecurity`에서 1개이다.

![security-filter-chain](/assets/images/spring-security/security-filter-chain.png)

- `WebSecurityConfigurerAdapter`가 여러개를 사용한다면, SecurityFilterChain 역시 많이 등록될 것이고, 순서는 @Order에 따라서 다르게
적용이 된다.

### VirtualFilterChain

```java
private static class VirtualFilterChain implements FilterChain {
    ...
	@Override
	public void doFilter(ServletRequest request, ServletResponse response)
		throws IOException, ServletException {
      if (currentPosition == size) {
        if (logger.isDebugEnabled()) {
          logger.debug(UrlUtils.buildRequestUrl(firewalledRequest)
						+ " reached end of additional filter chain; proceeding with original chain");
        }
        // Deactivate path stripping as we exit the security filter chain
			this.firewalledRequest.reset();
				originalChain.doFilter(request, response);
      }
      else {
        currentPosition++;
        Filter nextFilter = additionalFilters.get(currentPosition - 1);

        if (logger.isDebugEnabled()) {
          logger.debug(UrlUtils.buildRequestUrl(firewalledRequest)
						+ " at position " + currentPosition + " of " + size
						+ " in additional filter chain; firing Filter: '"
						+ nextFilter.getClass().getSimpleName() + "'");
        }

        nextFilter.doFilter(request, response, this);
      }
    }
}

```
- `FilterChainProxy`에서 Filters를 가져오고 실제 동작은 `VirtualFilterChain`에서 진행 된다.
- `VirtualFilterChain`에서 doFilter시에 this를 넘겨주고 있어 실행되는 Filter에서 또 `VirtualFilterChain`의 doFilter를 호출하여
연속적으로 호출이 되는 방식으로 설정되어 있는 Filters를 전체 실행하도록 되어 있다.
- `VirtualFilterChain.doFilter` -> `nextFilter.doFilter` -> `VirtualFilterChain.doFilter` -> `nextFilter.doFilter` -> ...
 

## 마치며
- `FilterChainProxy`는 Request마다 원하는 Filter Set을 URI를 보고 셋팅해주고, 실제로는 `WebSecurityConfigurerAdapter`
configure에서 Filter Set을 등록하게 된다.
- Security에서 Filter가 자동으로 만들어지는게 생각보다 많아서 나중에 각 Filter마다 역할을 알아보면 더 좋을 것 같다!

