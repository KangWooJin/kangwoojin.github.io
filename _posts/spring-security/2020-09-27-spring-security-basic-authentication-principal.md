---
title: "[Spring Security] Spring Security Basic - AuthenticationPrincipal" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-09-27 22:00:00+09:00
excerpt: AuthenticationPrincipal이 무엇인지 어떻게 동작하는지 알아보자.
---

## 들어가며
`@AuthenticationPrincipal`의 역할과 어떻게 동작하는지 알아보자.

## 어떤 역할을 하나?
```java
@GetMapping("/")
public String index(Model model, Principal principal) {
  if (principal != null) {
    model.addAttribute("message", "Hello Spring " + principal.getName());
  } else {
    model.addAttribute("message", "Hello Spring Security");
  }
  return "index";
}
```
- `Principal` 객체를 api parameter로 받을 수 있는데 `Principal`이라는 클래스 말고 사용자가 원하는 클래스로
받을 수 있게 해주는 ArgumentResolver를 동작하게 해주는 Anootation 이다.

```java
public class UserAccount extends User {
  private Account account;

  public UserAccount(Account account) {
    super(account.getUsername(), account.getPassword(), List.of(new SimpleGrantedAuthority("ROLE_" + account.getRole())));
    this.account = account;
  }

  public Account getAccount() {
    return account;
  }
}
```

```java
@GetMapping("/")
public String index(Model model, @AuthenticationPrincipal UserAccount userAccount) {
  if (userAccount != null) {
    model.addAttribute("message", "Hello Spring " + userAccount.getUsername());
  } else {
    model.addAttribute("message", "Hello Spring Security");
  }
  return "index";
}
```
- `@AuthenticationPrincipal`을 적용하면 `User`를 상속 받은 `UserAccount`를 사용할 수 있다.

```java
@Override
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
  final Account account = accountRepository.findByUsername(username);
  if (account == null) {
    throw new UsernameNotFoundException(username);
  }
  return new UserAccount(account);
}
```
- `UserDetailsService`에 loadUserByUsername 부분에 UserAccount를 전달하는거 까지 수정해야 정상적으로 동작한다.
- 여기까지 수정해야 하는 이유에 대해서 알아보자.

## 어떻게 동작하나?

- `AuthenticationPrincipalArgumentResolver`에 대해서 설명하기 앞서 어떻게 해당 resolver가 등록되는지 알아보자.

```java
@Retention(value = java.lang.annotation.RetentionPolicy.RUNTIME)
@Target(value = { java.lang.annotation.ElementType.TYPE })
@Documented
@Import({ WebSecurityConfiguration.class,
		SpringWebMvcImportSelector.class,
		OAuth2ImportSelector.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableWebSecurity {
    ...
  boolean debug() default false;
}
```
- `@EnableWebSecurity`을 사용하게 되면 `SpringWebMvcImportSelector`가 Import 된다.

```java
class SpringWebMvcImportSelector implements ImportSelector {
  public String[] selectImports(AnnotationMetadata importingClassMetadata) {
    boolean webmvcPresent = ClassUtils.isPresent(
      "org.springframework.web.servlet.DispatcherServlet",
			getClass().getClassLoader());
    return webmvcPresent
	  ? new String[] {
	    "org.springframework.security.config.annotation.web.configuration.WebMvcSecurityConfiguration" }
	    : new String[] {};
  }
}
```
- `SpringWebMvcImportSelector`에서 webMvc인 경우 `WebMvcSecurityConfiguration`를 추가하게 된다.

```java
class WebMvcSecurityConfiguration implements WebMvcConfigurer, ApplicationContextAware {
  private BeanResolver beanResolver;

  @Override
  @SuppressWarnings("deprecation")
  public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
    AuthenticationPrincipalArgumentResolver authenticationPrincipalResolver = new AuthenticationPrincipalArgumentResolver();
    authenticationPrincipalResolver.setBeanResolver(beanResolver);
    argumentResolvers.add(authenticationPrincipalResolver);
    argumentResolvers
		.add(new org.springframework.security.web.bind.support.AuthenticationPrincipalArgumentResolver());

    CurrentSecurityContextArgumentResolver currentSecurityContextArgumentResolver = new CurrentSecurityContextArgumentResolver();
    currentSecurityContextArgumentResolver.setBeanResolver(beanResolver);
    argumentResolvers.add(currentSecurityContextArgumentResolver);
    argumentResolvers.add(new CsrfTokenArgumentResolver());
  }
  ...
}
```
- `WebMvcSecurityConfiguration`에 `addArgumentResolvers`에 의해서 `AuthenticationPrincipalArgumentResolver`가 등록이 된다.
- `AuthenticationPrincipalArgumentResolver`은 `@AuthenticationPrincipal`을 이용해서 `Principal`을 가져오는 역할을 담당한다.
- `CurrentSecurityContextArgumentResolver`는 `@CurrentSecurityContext`을 이용해서 `SecurityContext`을 가져오는 역할을 담당한다.

```java
public final class AuthenticationPrincipalArgumentResolver
		implements HandlerMethodArgumentResolver {

	private ExpressionParser parser = new SpelExpressionParser();

	private BeanResolver beanResolver;

	public boolean supportsParameter(MethodParameter parameter) {
		return findMethodAnnotation(AuthenticationPrincipal.class, parameter) != null;
	}
	public Object resolveArgument(MethodParameter parameter,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest,
			WebDataBinderFactory binderFactory) {
		Authentication authentication = SecurityContextHolder.getContext()
				.getAuthentication();
		if (authentication == null) {
			return null;
		}
		Object principal = authentication.getPrincipal();

		AuthenticationPrincipal authPrincipal = findMethodAnnotation(
				AuthenticationPrincipal.class, parameter);

		String expressionToParse = authPrincipal.expression();
		if (StringUtils.hasLength(expressionToParse)) {
			StandardEvaluationContext context = new StandardEvaluationContext();
			context.setRootObject(principal);
			context.setVariable("this", principal);
			context.setBeanResolver(beanResolver);

			Expression expression = this.parser.parseExpression(expressionToParse);
			principal = expression.getValue(context);
		}

		if (principal != null
				&& !parameter.getParameterType().isAssignableFrom(principal.getClass())) {

			if (authPrincipal.errorOnInvalidType()) {
				throw new ClassCastException(principal + " is not assignable to "
						+ parameter.getParameterType());
			}
			else {
				return null;
			}
		}
		return principal;
	}
    ...
}
```
- `AuthenticationPrincipalArgumentResolver`에서 `Principal`을 가져와서 파라메터로 주입을 해주는데, `SecurityContextHolder`에 있는
`SecurityContext` 값에 `Principal`을 가져와서 주입해주는 역할을 한다.
- 해당 부분이 가능한 이유는 `Filter`가 `ArgumentResolver`보다 동작이 빠르기 때문에 인증된 `SecurityContext`를 가져올 수 있다.
- `Expression`이 존재하는데, `Expression`을 이용하면, 좀 더 디테일한 객체까지 가져올 수 있게 된다.

```java
@GetMapping("/")
public String index(Model model, @AuthenticationPrincipal(expression = "#this == 'anonymousUser' ? null : account") Account account) {
  if (account != null) {
    model.addAttribute("message", "Hello Spring " + account.getUsername());
  } else {
    model.addAttribute("message", "Hello Spring Security");
  }
  return "index";
}
```
- 예를들어 `@AuthenticationPrincipal(expression = "#this == 'anonymousUser' ? null : account")`라고 한다면, anonymousUser가 아니라면
`UserAccount`에 `Account` 객체를 주입받을 수 있게 된다.


## 마치며
- `@EnableWebSecurity`을 사용하는거랑 `@Configuration`을 사용하는거랑 똑같이 동작하기에 무엇이 다른가 싶었는데,
`SpringWebMvcImportSelector`에 의해서 주입되는 `WebMvcSecurityConfiguration`에 차이가 있었다.
- 처음부터 끝까지 강의 영상을 거의 다 봤는데, 끝에 가니 모든 퍼즐이 맞춰지는 느낌이다. 이제 얼마 안남았는데 마지막까지 다 들을 수 있으면 좋겠다!!