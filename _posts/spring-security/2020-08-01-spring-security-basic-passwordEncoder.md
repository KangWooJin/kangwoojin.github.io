---
title: "[Spring Security] Spring Security Basic - PasswordEncoder" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-08-01 09:00:00+09:00 
excerpt: PasswordEncoder의 역할은 무엇이고, 언제 동작하는지 알아보자.
---

## 들어가며
PasswordEncoder의 역할은 무엇이고, 언제 동작하는지 알아보자.

1. PasswordEncoder의 역할은 무엇인가?
2. PasswordEncoder는 언제 등록 되는가?
3. PasswordEncoder는 언제 동작하는가?

## PasswordEncoder의 역할은 무엇인가?
- `PasswordEncoder`는 password를 암호화해서 사용자의 password를 더 안전하게 하기 위한 용도로 사용 된다.
  
## PasswordEncoder는 언제 등록 되는가?
- `InitializeUserDetailsBeanManagerConfigurer`에서 `init`후 `InitializeUserDetailsManagerConfigurer`이 configure될 때
 `UserDetailsService`가 존재한다면 `PasswordEncoder`를 등록하게 되어 있다.

```java
public void configure(AuthenticationManagerBuilder auth) throws Exception {
  if (auth.isConfigured()) {
    return;
  }
  UserDetailsService userDetailsService = getBeanOrNull(
    UserDetailsService.class);
  if (userDetailsService == null) {
    return;
  }

  PasswordEncoder passwordEncoder = getBeanOrNull(PasswordEncoder.class);
  UserDetailsPasswordService passwordManager = getBeanOrNull(UserDetailsPasswordService.class);

  DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
  provider.setUserDetailsService(userDetailsService);
  if (passwordEncoder != null) {
    provider.setPasswordEncoder(passwordEncoder);
  }
  if (passwordManager != null) {
    provider.setUserDetailsPasswordService(passwordManager);
  }
  provider.afterPropertiesSet();
  auth.authenticationProvider(provider);
}
```

- `PasswordEncoder`는 Security에서 기본적으로 등록해주는 것이 없기에 Bean으로 직접 등록하지 않으면 null로 셋팅이 된다.
- 만약 `PasswordEncoder`가 없이 password를 사용한다면, `DelegatingPasswordEncoder`에서 passwordEncoder를 찾지 못해
`UnmappedIdPasswordEncoder`에 matches가 실행되어 에러가 발생 한다.

```java
@Override
public boolean matches(CharSequence rawPassword, String prefixEncodedPassword) {
  if (rawPassword == null && prefixEncodedPassword == null) {
    return true;
  }
  String id = extractId(prefixEncodedPassword);
  PasswordEncoder delegate = this.idToPasswordEncoder.get(id);
  if (delegate == null) {
    return this.defaultPasswordEncoderForMatches
			.matches(rawPassword, prefixEncodedPassword);
  }
  String encodedPassword = extractEncodedPassword(prefixEncodedPassword);
  return delegate.matches(rawPassword, encodedPassword);
}
``` 

- password가 encoder에 의해서 encode 되면, `{알고리즘}password`형태의 password로 저장되게 된다.
- `{알고리즘}` 부분을 extract하여 encoder를 찾게되는데, encoder가 없어 `delegate == null`로 되어 버린다.
- 따라서 `defaultPasswordEncoderForMatches`의 로직을 타는데 default는 `UnmappedIdPasswordEncoder` 해당 encoder로 되어 있다.

```java
private class UnmappedIdPasswordEncoder implements PasswordEncoder {
  @Override
  public String encode(CharSequence rawPassword) {
    throw new UnsupportedOperationException("encode is not supported");
  }
  @Override
  public boolean matches(CharSequence rawPassword, String prefixEncodedPassword) {
    String id = extractId(prefixEncodedPassword);
    throw new IllegalArgumentException("There is no PasswordEncoder mapped for the id \"" + id + "\"");
  }
}
```

- `UnmappedIdPasswordEncoder`의 메소드가 호출되는 경우 에러가 발생하게 되어 있으니.. `passwordEncoder`가 등록이 잘 되었는지 확인하는 것도 중요하다.
### PasswordEncoder 등록하기

```java
@Bean
public PasswordEncoder passwordEncoder() {
  return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}
```

- Spring Security에서 다양한 PasswordEncoder를 `PasswordEncoderFactories`에서 제공해주고 있다.
- spring 5로 넘어가면서 default passwordEncoder 값은 `bcrypt`로 encode 하지만 여러 PasswordEncoder를 등록해두어서,
password의 알고리즘타입에 의해 matches를 실행한다.
   
## PasswordEncoder는 언제 동작하는가?
- `DaoAuthenticationProvider.authenticate` 메소드 안에 `additionalAuthenticationChecks` 해당 메소드를 실행하면서
`passwordEncoder.matches`를 통해서 입력받은 password와 저장되어 있는 password를 비교한다.

```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
    ...
	try {
	  preAuthenticationChecks.check(user);
	  additionalAuthenticationChecks(user,
				(UsernamePasswordAuthenticationToken) authentication);
	}
	catch (AuthenticationException exception) {
	  if (cacheWasUsed) {
	    // There was a problem, so try again after checking
		// we're using latest data (i.e. not from the cache)
		cacheWasUsed = false;
		user = retrieveUser(username,
				(UsernamePasswordAuthenticationToken) authentication);
		preAuthenticationChecks.check(user);
		additionalAuthenticationChecks(user,
				(UsernamePasswordAuthenticationToken) authentication);
	  }
	  else {
	    throw exception;
	  }
	}
	...
}
```

```java
protected void additionalAuthenticationChecks(UserDetails userDetails,
		UsernamePasswordAuthenticationToken authentication)
		throws AuthenticationException {
  if (authentication.getCredentials() == null) {
    logger.debug("Authentication failed: no credentials provided");

    throw new BadCredentialsException(messages.getMessage(
      "AbstractUserDetailsAuthenticationProvider.badCredentials",
				"Bad credentials"));
  }

  String presentedPassword = authentication.getCredentials().toString();

  if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
    logger.debug("Authentication failed: password does not match stored value");

    throw new BadCredentialsException(messages.getMessage(
      "AbstractUserDetailsAuthenticationProvider.badCredentials",
				"Bad credentials"));
  }
}
```


