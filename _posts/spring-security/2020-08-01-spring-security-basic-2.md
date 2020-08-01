---
title: "[Spring Security] Spring Security Basic - UserDetailsService" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-08-01 09:00:00+09:00 
excerpt: UserDetailsService가 언제 동작하고, 왜 동작하는지 알아보자.
---

## 들어가며
SpringSecurity에서 제공하는 UserDetailsService 인터페이스에 대해서 알아본다.

1. UserDetailsService의 역할은 무엇인가?
2. UserDetailsService는 언제 동작하는가?
3. UserDetailsService가 언제 등록 되는가?

## UserDetailsService의 역할은 무엇인가?

```java
public interface UserDetailsService {
	UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

- UserDetailsService는 method 하나를 가지고 있는 interface이다.
- login form에서 입력한 username을 이용해서 UserDetails 객체를 생성해주는 역할을 한다.
- 해당 인터페이스를 상속받고 `Bean`으로 등록하게 되면, authentication을 할때 사용되게 된다.

### 사용 예제
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
- Account 관련된 Service에 상속하여, username 기반으로 Account 정보를 조회 후 SpringSecurity에서 제공하는
`User.Builder`를 통해서 UserDetails를 만들어서 제공하면 된다.
- 여기서 `Role`은 `Authorization`에 사용 된다.

## UserDetailsService 는 언제 동작하는가?
- login 요청시에 해당 User에 대한 정보를 가져오기 위해서 호출이 발생한다.
- 해당 부분을 좀 더 자세히 알아보자.

### UsernamePasswordAuthenticationFilter
```java
public Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException {
    if (postOnly && !request.getMethod().equals("POST")) {
      throw new AuthenticationServiceException(
        "Authentication method not supported: " + request.getMethod());
    }

	String username = obtainUsername(request);
	String password = obtainPassword(request);
	if (username == null) {
		username = "";
	}

	if (password == null) {
			password = "";
	}

	username = username.trim();

	UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
			username, password);
	// Allow subclasses to set the "details" property
	setDetails(request, authRequest);

	return this.getAuthenticationManager().authenticate(authRequest);
}
```

- form창에서 입력한 user 정보를 `UsernamePasswordAuthenticationToken`에 셋팅하고
`AuthenticationManager`에 authenticate를 요청한다.
- 이 부분을 확인하기 위해서는 debug point를 찍고 확인이 필요하다.

### ProviderManager

![providerManager](/assets/images/spring-security/providerManager.png)

- `ProviderManager`는 `AuthenticationManager`를 상속받고 있는 class이다.
- `ProviderManager`는 여러 `AuthenticationProvider`를 갖고 있어서 Provider를 통해서 authentication을 진행해
result를 return 한다.
- result는 유저 정보를 의미한다.

```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {    
    Class<? extends Authentication> toTest = authentication.getClass();
    ...
    
    for (AuthenticationProvider provider : getProviders()) {
      if (!provider.supports(toTest)) {
        continue;
      }
      
      try {
        // 해당 부분에서 User 정보를 가져 온다.
        result = provider.authenticate(authentication);
        if (result != null) {
          copyDetails(authentication, result);
          break;
        }
      } catch (AccountStatusException | InternalAuthenticationServiceException e) {
        prepareException(e, authentication);
        // SEC-546: Avoid polling additional providers if auth failure is due to
		// invalid account status
		throw e;			
      }
     catch (AuthenticationException e) {
        lasException = e;
			
     }
    ...
}
```
 
- `AnonymousAuthenticationProvider`를 먼저 검사 후, `AnonymousAuthenticationProvider`의 parent인 `DaoAuthenticationProvider.authenticate`이 부분이 실행 된다.
 
### DaoAuthenticationProvider
- `DaoAuthenticationProvider`는 `AbstractUserDetailsAuthenticationProvider`의 구현 class이고,
`retrieveUser`부분만 추상화 되어 있다.
- `DaoAuthenticationProvider.authenticate`에 `retrieveUser` 부분이 호출되면, `UserDetailsService.loadUserByUsername`을 
통해서 User 를 가져오게 되어 있다.
   
```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
        ...
	if (user == null) {
	  cacheWasUsed = false;
	  try {
	    user = retrieveUser(username,
				(UsernamePasswordAuthenticationToken) authentication);
	  }
	  catch (UsernameNotFoundException notFound) {
	    logger.debug("User '" + username + "' not found");
	    if (hideUserNotFoundExceptions) {
	      throw new BadCredentialsException(messages.getMessage(
	        "AbstractUserDetailsAuthenticationProvider.badCredentials",
						"Bad credentials"));
	    }
	    else {
	      throw notFound;
	    }
	  }
	  ...
	}
    ...
}
```

```java
protected final UserDetails retrieveUser(String username,
			UsernamePasswordAuthenticationToken authentication)
			throws AuthenticationException {
  prepareTimingAttackProtection();
  try {
    UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
    if (loadedUser == null) {
      throw new InternalAuthenticationServiceException(
        "UserDetailsService returned null, which is an interface contract violation");
    }
    return loadedUser;
  }
  catch (UsernameNotFoundException ex) {
    mitigateAgainstTimingAttack(authentication);
    throw ex;
  }
  catch (InternalAuthenticationServiceException ex) {
    throw ex;
  }
  catch (Exception ex) {
    throw new InternalAuthenticationServiceException(ex.getMessage(), ex);
  }
}
```

- `ProviderManager`에 등록되어 있는 `AuthenticationProvider`중에서 User 정보를 읽어서 확인하는
과정에 `UserDetailsService`가 사용이 되는 것을 확인할 수 있다.

## UserDetailsService가 언제 등록 되는가?
- `InitializeUserDetailsBeanManagerConfigurer`에서 `UserDetilasService`가 Bean으로 등록되어 있는 경우
`DaoAuthenticationProvider`에 `userDetailsService`로 등록 된다.

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

```java
private <T> T getBeanOrNull(Class<T> type) {
  String[] beanNames = InitializeUserDetailsBeanManagerConfigurer.this.context
			.getBeanNamesForType(type);
  if (beanNames.length != 1) {
    return null;
  }

  return InitializeUserDetailsBeanManagerConfigurer.this.context
			.getBean(beanNames[0], type);
}
```

- 2개 이상의 UserDetailsService가 등록되어 있는 경우, 어떤 것을 사용해야 할지 모르기 때문에
`DaoAuthenticationProvider`에 userDetailsService가 등록되지 않게 된다.