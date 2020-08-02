---
title: "[Spring Security] Spring Security Basic - Security test" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-02 09:00:00+09:00 
excerpt: Spring Security관련  테스트를 하는 방법에 대해서 알아보자~
---

## 들어가며
- Spring Security를 적용하고, 매번 web을 이용해서 테스트 해보는 방식은 매우 불편하다.
- 서버를 매번 기동하고, 계정을 만들고 테스트를 진행하는 방식이 아니라, spring security test를 이용해서
간단하게 원하는 인증과 인가에 대해서 테스트를 진행하는 방법에 대해서 알아보자.

## Spring Security Test
```groovy
dependencies {
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
		exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
	}
    testImplementation 'org.springframework.security:spring-security-test'
}
``` 
- Spring Security Test를 편하게 하기 위해서 Security용 테스트를 따로 제공해주고 있다.
- Spring Security Test와 Spring Test를 추가해서 테스트 환경을 구축 한다.

## MockUser
- MockUser는 인증된 유저가 있다고 가정하고, 인가(Authorization) 부분을 쉽게 테스트할 수 있게 도와 준다.
- 테스트를 진행하는 방법은 2가지로, method chain 방식과 annotation 방식이 존재한다.
 
### Method chain By User

```java
@Test
void indexUser() throws Exception {
  mockMvc.perform(get("/").with(user("woojin").roles("USER")))
         .andExpect(status().isOk());
}

// Annotation 방식
@Test
@WithMockUser(username = "admin", roles = "USER")
void indexUser() throws Exception {
  mockMvc.perform(get("/"))
         .andExpect(status().isOk());
}
```

- spring에서 제공해주는 test와 spring security에서 제공해주는 test를 합쳐서 mockMvc 테스트를 할때 유저 정보를
mocking 할 수 있다.
- `with.(user("username").password("password).roles("role"))` 처럼 chain으로 연결해서 생성이 가능하다.

### Method chain By Anonymous

```java
@Test
void indexAnonymous() throws Exception {
  mockMvc.perform(get("/").with(anonymous()))
         .andExpect(status().isOk());
}

// Annotation 방식
@Test
@WithAnonymousUser
void indexAnonymous() throws Exception {
  mockMvc.perform(get("/"))
         .andExpect(status().isOk());
}
```

- Spring security test에서는 인증되지 않은 유저에 대해서 테스트를 하는 방법도 제공해준다. 
- anonymous의 경우는 `user()` 대신 `annoymous()`를 사용해서 테스트를 진행하면 된다.

## Authenticate

```java
@Test
void loginSuccess() throws Exception {
  String username = "woojin";
  String password = "123";
  Account user = getAccount(username, password);
  mockMvc.perform(formLogin().user(user.getUsername()).password(password))
         .andExpect(authenticated());
}

@Test
void loginFail() throws Exception {
    String username = "woojin";
    String password = "123";
    Account user = getAccount(username, password);
    mockMvc.perform(formLogin().user(user.getUsername()).password("empty"))
           .andExpect(unauthenticated());
}

private Account getAccount(String username, String password) {
  Account user = Account.builder()
                        .username(username)
                        .password(password)
                        .role("USER")
                        .build();
  accountService.createNew(user);
  return user;
}
```

- formLogin에 대해서 테스트를 진행할때 Spring Security Test에서 제공하는 `formLogin`을 이용해서 확인할 수 있다.
- mock으로 제공할 user 정보로 실제 login이 가능한지 확인한다. 이때, UserDetailsService에 구현에 따라서 반영되기에 
해당 유저가 실제 존재하는지 확인이 필요하다.
- 해당 테스트를 진행할 때는 `@WithMockUser`로 Mock처리 할 수 없어, formLogin()에 builder를 이용해서 값을 채워줘야 한다.

