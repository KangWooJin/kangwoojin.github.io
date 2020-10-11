---
title: "[Spring Security] Spring Security Basic - GlobalMethodSecurityConfiguration" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-10-11 14:00:00+09:00
excerpt: GlobalMethodSecurityConfiguration의 역할이 무엇이고 어떻게 동작하고 있는지 알아보자.
---

## 들어가며
- `GlobalMethodSecurityConfiguration`는 Method level에 Security를 설정할 수 있게 해준다.
- 즉, web service가 아니더라도 security를 적용하여 ROLE 단위에 사용가능한 method를 설정할 수 있다.

## 어떤 역할을 하는가?
```java
@EnableGlobalMethodSecurity(securedEnabled = true, prePostEnabled = true, jsr250Enabled = true)
public class MethodSecurity extends GlobalMethodSecurityConfiguration {

}
```
- `@EnableGlobalMethodSecurity`를 통해서 enable 상태로 변경해야 한다.
- 그 중에서 사용하고 싶은 방법 중 원하는 것을 선택해서 활성화 하고 사용하면 된다.
- 추가로 설정을 하고 싶다면 `GlobalMethodSecurityConfiguration`을 상속해서 오버라이드해서 사용하면 된다.
- 총 `@Secured`, [`@PreAuthorize`, `@PostAuthorize`], `@RolesAllowed`로 3가지 방법이 존재한다.
  
### JSR-250 방식
- JSR-250 방식은 `javax.annotation.security`에 존재하는 annotation을 이용해서 security를 설정하는 방식이다.
- 3가지 annotation을 사용할 수 있는데 `@PermitAll`, `@DenyAll`, `@RolesAllowed` 방식이 존재한다.

```java
@RolesAllowed({"ROLE_USER","ROLE_ADMIN"})
//@PermitAll
//@DenyAll
public void dashboard() {
  Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
  Object principal = authentication.getPrincipal();
  Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
  Object credentials = authentication.getCredentials();
  boolean authenticated = authentication.isAuthenticated();
}
```
- 해당 방식처럼 method에 annotation을 설정해서 사용하면 된다.
- PermitAll, DenyAll은 말 그대로 전체허용, 거부 이고, RolesAllowed는 value에 전달된 ROLE을 검사해서 확인하게 된다.
- `Jsr250Voter`에서 `PermitAll`, `DenyAll`에 대해서 확인 후 처리 한다.

### secured 방식
```java
public interface BankService {
  @Secured("IS_AUTHENTICATED_FULLY")
  public Account readAccount(Long id);
  @Secured("IS_AUTHENTICATED_ANONYMOUSLY")
  public Account[] findAccounts();
  @Secured({"ROLE_USER", "ROLE_ADMIN"})
  public Account post(Account account, double amount);
}
```
- `@Secured`를 이용해서 사용하면 되고, ROLE단위에 매칭도 가능하고 좀 더 다양한 기능을 지원한다.
- 해당 방식은 `RoleVoter`, `AuthenticatedVoter`에서 값을 확인하고 처리하게 된다.
- `RoleVoter`는 `ROLE_`로 시작하는 경우 확인하게 되고, `AuthenticatedVoter`에서 `IS_AUTHENTICATED_FULLY`, `IS_AUTHENTICATED_ANONYMOUSLY`, `IS_AUTHENTICATED_REMEMBERED`에 대해서만
검증을 한다.

### prePost 방식

```java
public interface BankService {
  @PreAuthorize("isAnonymous()")
  public Account readAccount(Long id);

  @PreAuthorize("isAnonymous()")
  public Account[] findAccounts();

  @PreAuthorize("hasAuthority('ROLE_TELLER')")
  public Account post(Account account, double amount);
}
```
- `@PreAuthorize`, `@PostAuthorize`를 통해서 method가 실행되기 전, 실행 후에 authorize를 확인할 수 있다.
- 앞에 방식보다 좀 더 Spel을 사용할 수 있고, `hasRole('ROLE_USER')` 처럼 사용해서 ROLE 단위에 매칭도 사용 가능하다.

```java
@PostAuthorize
  ("returnObject.username == authentication.principal.nickName")
public CustomUser loadUserDetail(String username) {
    return userRoleRepository.loadUserByUserName(username);
}

@PreAuthorize("#username == authentication.principal.username")
public String getMyRoles(String username) {
    //...
}
```
- returnObject와 paramater를 바탕으로도 설정을 제한할 수 있어서 좀 더 다양하게 활용될 수 있다.

```java
@EnableGlobalMethodSecurity(securedEnabled = true, prePostEnabled = true, jsr250Enabled = true)
public class MethodSecurity extends GlobalMethodSecurityConfiguration {
//    @Override
//    protected AccessDecisionManager accessDecisionManager() {
//        RoleHierarchyImpl roleHierarchy = new RoleHierarchyImpl();
//        roleHierarchy.setHierarchy("ROLE_ADMIN > ROLE_USER");
//        AffirmativeBased accessDecisionManager = (AffirmativeBased) super.accessDecisionManager();
//        accessDecisionManager.getDecisionVoters().add(new RoleHierarchyVoter(roleHierarchy));
//
//        return accessDecisionManager;
//    }

    @Override
    protected MethodSecurityExpressionHandler createExpressionHandler() {
        final DefaultMethodSecurityExpressionHandler handler = new DefaultMethodSecurityExpressionHandler();
        final RoleHierarchyImpl roleHierarchy = new RoleHierarchyImpl();
        roleHierarchy.setHierarchy("ROLE_ADMIN > ROLE_USER");
        handler.setRoleHierarchy(roleHierarchy);
        return handler;
    }
}
```
- `PreInvocationAuthorizationAdviceVoter`에서만 `MethodSecurityExpressionHandler`를 이용해서 `RoleHierarchy`를 적용할 수 있다.
- `GlobalMethodSecurityConfiguration`를 상속해서 method를 override하고 expressionHandler에 RoleHierarchy를 설정한다.
- 해당 방식으로 하게 되면 Pre, Post에서만 사용가능 하나, 주석처리되어 있는 방식으로 하게 되면 AccessDecisionManager에 voter를 추가하기 때문에
설정과 무관하게 roleHierarchy를 적용할 수 있다.

## 마치며
- 대부분 web security를 활용해서 제한을 많이 하지만, 만약 PC용 프로그램에 제한을 걸어야 한다면 method security를 이용해서 제한을 걸어도 괜찮아보인다.
- 그 외에도 request matcher 방식이 아닌 annotation 방식을 이용해서 제한을 하고 싶을 때도 사용이 가능해 보여서 알아두면 좋은 것 같다.

