---
title: "[Spring Security] Spring Security Basic - Configuration" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-01 12:00:00+09:00 
excerpt: Spring Security의 Configuration이 어떻게 동작하는지, 어떻게 설정하는지에 대해서 알아보자.
---

## 들어가며
Spring security를 적용했을 때 변경되는 부분과, 어떻게 하면 원하는 부분에 security를 적용할 수 있는지에
대해서 알아보자.

1. Spring security dependency만 추가했는데 Spring Security가 적용 되는 이유
2. `WebSecurityConfigurerAdapter`를 통해서 설정 값을 변경하는 방법
3. `Authentication`을 적용하는 방법

## Spring security dependency만 추가했는데 Spring Security가 적용 되는 이유
### Spring Security AutoConfiguration

- Spring Security 관련 Dependency를 해당 프로젝트에 추가 해본 뒤에 web server를 띄운다면
모든 요청에 대해서 login 페이지가 나와서 요청 제한이 걸리는 것을 알 수 있다.
- 아무런 설정을 추가하지 않았는데 어떻게 이런일이 발생하는지 확인을 해보자~

### Spring Security Setting
- 해당 부분을 테스트 해보기 위해서 Spring Security example을 만들어보자.

```groovy
dependencies {
	implementation 'org.springframework.boot:spring-boot-starter-security'
	implementation 'org.springframework.boot:spring-boot-starter-web'
    ...

	compileOnly 'org.projectlombok:lombok'
	annotationProcessor 'org.projectlombok:lombok'
	testImplementation('org.springframework.boot:spring-boot-starter-test') {
		exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
	}
	testImplementation 'org.springframework.security:spring-security-test'
}
```

- 가장 중요한 부분이 web, security를 추가 후 spring boot를 기동 후 `/` page에 접근하면 login 페이지로
redirect 되는 것을 알 수 있다.

![spring-security-login](/assets/images/spring-security/spring-security-login.png)

### SecurityAutoConfiguration


```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(DefaultAuthenticationEventPublisher.class)
@EnableConfigurationProperties(SecurityProperties.class)
@Import({ SpringBootWebSecurityConfiguration.class, WebSecurityEnablerConfiguration.class,
		SecurityDataConfiguration.class })
public class SecurityAutoConfiguration {
	@Bean
	@ConditionalOnMissingBean(AuthenticationEventPublisher.class)
	public DefaultAuthenticationEventPublisher authenticationEventPublisher(ApplicationEventPublisher publisher) {
		return new DefaultAuthenticationEventPublisher(publisher);
	}

}
```

- 이유는 SpringBoot에서 제공해주는 `SecurityAutoConfiguration`에서 `@Import`에 
`SpringBootWebSecurityConfiguration`가 포함되어 있기 때문이다.


```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(WebSecurityConfigurerAdapter.class)
@ConditionalOnMissingBean(WebSecurityConfigurerAdapter.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
public class SpringBootWebSecurityConfiguration {

	@Configuration(proxyBeanMethods = false)
	@Order(SecurityProperties.BASIC_AUTH_ORDER)
	static class DefaultConfigurerAdapter extends WebSecurityConfigurerAdapter {

	}
}
```

- `SpringBootWebSecurityConfiguration`에서는 `@ConditionalOnMissingBean(WebSecurityConfigurerAdapter.class)` 일때
`DefaultConfigurerAdapter`를 생성하게 된다.
- Dependency만 추가하게 되면, `WebSecurityConfigurerAdapter`를 bean으로 등록하는 로직이 없기때문에
`DefaultConfigurerAdapter`가 등록이 되어서 default Security가 적용이 된다.
 
```java
protected void configure(HttpSecurity http) throws Exception {
  logger.debug("Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).");
  http.authorizeRequests()
	  .anyRequest().authenticated()
	  .and()
	  .formLogin().and()
	  .httpBasic();
}
```
- `DefaultConfigurerAdapter`에 http configure 설정이다.
- 모든 요청에 대해서 인증을 받아야 하고, 인증이 없는 경우 `formLogin`을 하며, `httpBasic`을 이용하게 
기본 셋팅이 들어가게 되어 있다.
 
## `WebSecurityConfigurerAdapter`를 통해서 설정 값을 변경하는 방법
- 실제 사용할때는 default를 사용하는 것이 아닌 원하는 설정을 해야 하니, `SecurityConfig`을 
만들어 변경해보자~

```java
@EnableWebSecurity(debug = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .mvcMatchers("/", "/info", "/account/**").permitAll()
            .mvcMatchers("admin").hasAnyRole("ADMIN")
            .anyRequest().authenticated();

        http.formLogin();
        http.httpBasic();
    }
}
```

- `WebSecurityConfigurerAdapter`를 `Bean`으로 등록하게 되면, Springboot에서 만들어주는 
`DefaultConfigurerAdapter`가 생성되지 않게 할 수 있다.
- 따라서 `WebSecurityConfigurerAdapter`을 상속 받은 `SecurityConfig`를 만들고, `@EnableWebSecurity`
annotation을 추가하여, `@Configuration`이 될 수 있도록 한다.
- 그 후 `protected void configure(HttpSecurity http)` 부분을 override 하여
원하는 request에 요청을 제한할 수 있다.

## Authentication 적용하는 방법
### SecurityProperties User
- Spring Security는 기본적으로 User를 만드는 로직이 포함되어 있는데, 어떻게 생성되는지 알아 보자.

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(AuthenticationManager.class)
@ConditionalOnBean(ObjectPostProcessor.class)
@ConditionalOnMissingBean(
		value = { AuthenticationManager.class, AuthenticationProvider.class, UserDetailsService.class },
		type = { "org.springframework.security.oauth2.jwt.JwtDecoder",
				"org.springframework.security.oauth2.server.resource.introspection.OpaqueTokenIntrospector" })
public class UserDetailsServiceAutoConfiguration {
    ...
	@Bean
	@ConditionalOnMissingBean(
			type = "org.springframework.security.oauth2.client.registration.ClientRegistrationRepository")
	@Lazy
	public InMemoryUserDetailsManager inMemoryUserDetailsManager(SecurityProperties properties,
			ObjectProvider<PasswordEncoder> passwordEncoder) {
		SecurityProperties.User user = properties.getUser();
		List<String> roles = user.getRoles();
		return new InMemoryUserDetailsManager(
				User.withUsername(user.getName()).password(getOrDeducePassword(user, passwordEncoder.getIfAvailable()))
						.roles(StringUtils.toStringArray(roles)).build());
	}
}
```

- `UserDetailsServiceAutoConfiguration`에 `@Conditional`관련 조건에 의해서 해당 `Configuration`이 
Bean으로 등록되면 `InMemoryUserDetailsManager`가 Bean으로 등록될 때 user가 등록되게 되어 있다.
- 해당 default user의 정보는 `SecurityProperties`의 기본 값에 의해서 `User` 값이 채워져서 생성이 된다.

```java
public static class User {
  private String name = "user";
  private String password = UUID.randomUUID().toString();
  private List<String> roles = new ArrayList<>();
  private boolean passwordGenerated = true;
  ...
  public void setPassword(String password) {
    if (!StringUtils.hasLength(password)) {
      return;
    }
    this.passwordGenerated = false;
    this.password = password;
  }
```

- password가 UUID로 random하게 생성되는데 어떻게 로그인을 할 수 있는가? 궁금할 수 있다.
- `passwordGenerated = true`로 생성된 경우, `InMemoryUserDetailsManager`에서 password를 등록할 때 console에 password를
찍어 주도록 되어 있어서 password를 확인할 수 있게 되어 있다.

```java
private String getOrDeducePassword(SecurityProperties.User user, PasswordEncoder encoder) {
  String password = user.getPassword();
  if (user.isPasswordGenerated()) {
    logger.info(String.format("%n%nUsing generated security password: %s%n", user.getPassword()));
  }
  if (encoder != null || PASSWORD_ALGORITHM_PATTERN.matcher(password).matches()) {
    return password;
  }
  return NOOP_PASSWORD_PREFIX + password;
}
```

- default user로만 사용하는 경우 User가 1개만 등록되고, 매번 Password가 바뀌게 되어 있어,
여러명의 유저와 원하는 유저를 등록하는 방법에 대해서 알아보자.

### InMemoryUserDetailsManager
- `InMemoryUserDetailsManager`에 `User` 정보를 등록하면 사용가능하다는 것을 앞에서 확인하였다.
- `InMemoryUserDetailsManager`는 `UserDetailsService`을 상속 받은 class이기에 `UserDetailsServiceAutoConfiguration`이
동작하지 않게 된다.
- `WebSecurityConfigurerAdapter`에 `protected void configure(AuthenticationManagerBuilder auth)` 부분을 overriding하여
유저 정보를 등록할 수 있다.

```java
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
  auth.inMemoryAuthentication()
      .withUser("woojin").password("{noop}123").roles("USER").and()
      .withUser("admin").password("{noop}admin").roles("ADMIN");
}
```

- 해당 방식은 서버가 가동될 때 유저 정보가 등록되기에 동적으로 유저 정보를 추가 할 수 없는 문제가 있다.
- 다음으로는 `User`정보를 database와 같은 저장소로부터 읽어들여, 인증 하는 방법을 확인해보자.
 
### UserDetailsService
- `UserDetailsService`가 Bean으로 등록하게 되면 `UserDetailsServiceAutoConfiguration`이 동작하지 않는것을 확인했었다.
- `UserDetailsService`는 method 하나를 갖고 있는 interface이다.
- `UserDetailsService`의 역할은 username을 받아서, `UserDetails`를 만들어 return 해주는 역할이다.

```java
public interface UserDetailsService {
	UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

- 해당 부분을 응용해서, username으로 user를 조회하고 UserDetails를 만드는 Service를 구현하면 된다.
```java
@Service
@RequiredArgsConstructor
public class AccountService implements UserDetailsService {
    private final AccountRepository accountRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        final Account account = accountRepository.findByUsername(username);
        if (account == null) {
            throw new UsernameNotFoundException(username);
        }
        return User.builder()
                   .username(account.getUsername())
                   .password(account.getPassword())
                   .roles(account.getRole())
                   .build();
    }
}
```

- 여기서 login form창에서 입력한 username을 전달 받아서 해당 username으로 database와 같은 저장소에 있는
유저를 가져와서 데이터를 셋팅해주면 된다.


## 마치며
- 백기선의 Spring Security를 보면서 정리한 내용이라서 이해하기 어려울 수 있다.. 까먹지 않기 위한 기록용이다..
- 궁금한 부분만 찾아보고 정리해보았는데.. 정확한 내용이 아닐 수 있으니 참고해야 한다..