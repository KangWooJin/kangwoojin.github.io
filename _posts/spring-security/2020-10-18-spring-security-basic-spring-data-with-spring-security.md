---
title: "[Spring Security] Spring Security Basic - SpringData With SpringSecurity" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-10-18 12:00:00+09:00
excerpt: SpringData에 SpringSecurity를 적용하는 방법에 대해서 알아보자.
---

## 들어가며
SpringData에서 Query를 사용할 때 Spel에 Principal를 이용하는 방법에 대해서 알아보자.

## 의존성 추가
```java
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-security'
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-data-jpa'

  // spring-security-data 추가
  implementation 'org.springframework.security:spring-security-data'

  runtimeOnly 'com.h2database:h2'

  compileOnly 'org.projectlombok:lombok'
  annotationProcessor 'org.projectlombok:lombok'
  testCompileOnly 'org.projectlombok:lombok'
  testAnnotationProcessor 'org.projectlombok:lombok'
  testImplementation('org.springframework.boot:spring-boot-starter-test') {
    exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
  }
  testImplementation 'org.springframework.security:spring-security-test'
}
```
- spring-security-data dependency를 추가하여, spring-data에서 security 관련 spel을 사용할 수 있게 설정한다.

## 사용법

```java
public interface AccountRepository extends JpaRepository<Account, Integer> {
  @Query("select a from Account a where a.username = ?#{principal?.username}")
  Account findByUsernameQuery();
}
```
- `Query`를 통해서 직접 사용할 때 `principal`을 가져와서 사용할 수 있다.
- `principal`은 `SecurityContextHolder.getContext().getAuthentication().getPrincipal()`와 같은 값을 사용 한다.
- 따라서 `Authentication`이며, 해당 객체에 있는 field값을 그대로 사용할 수 있다. 

## 테스트

```java
@Transactional
@AutoConfigureTestEntityManager
@SpringBootTest
class AccountRepositoryTest {

  @Autowired
  AccountRepository accountRepository;

  @Autowired
  TestEntityManager testEntityManager;

  @Test
  @WithMockUser(username = "woojin")
  void principalTest() {
    Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();
    assertThat(principal).isNotNull();
        
    Account account = new Account();
    account.setUsername("woojin");
    account.setPassword("123");
    account.setRole("USER");
    testEntityManager.persist(account);
    Account actual = accountRepository.findByUsernameQuery();
    assertThat(actual).isNotNull();
    assertThat(actual.getUsername()).isEqualTo("woojin");
    assertThat(actual.getRole()).isEqualTo("USER");
  }
}
```

- `WithMockUser`를 통해서 `Principal` 값에 mockUser를 주입하였다.
- 이후 Query가 잘 동작하는지 검증하였을때 저장했던 Account와 조회한 Account가 동일함을 확인하였다.

## 마치며
- 많이 사용하는 방법은 아니라 이러한 기능도 있구나 정도로만 생각하면 될거 같다.
