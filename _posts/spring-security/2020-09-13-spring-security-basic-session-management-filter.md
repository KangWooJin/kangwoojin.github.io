---
title: "[Spring Security] Spring Security Basic - SessionManagementFilter" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-09-13 14:00:00+09:00
excerpt: SessionManagementFilter가 어떤 역할을 하는지 알아보자.
---

## 들어가며
SessionManagementFilter가 어떤 역할을 하는지 알아보자.

## session 변조 방지 전략 설정
- 인증(로그인)을 할때 마다 session id를 변경하거나, session을 변경하여 session 정보가 노출 되더라도
새롭게 로그인 되었을 때 이전에 session이 재사용되는 것을 방지

### migrateSession
```java
http.sessionManagement().sessionFixation().migrateSession();
```
- 설정 방법은 `WebSecurityConfigurerAdapter`에 http configure에서 설정 가능하다.
 
```java
@Override
final HttpSession applySessionFixation(HttpServletRequest request) {
  HttpSession session = request.getSession();
  String originalSessionId = session.getId();
  if (logger.isDebugEnabled()) {
    logger.debug("Invalidating session with Id '" + originalSessionId + "' "
				+ (migrateSessionAttributes ? "and" : "without")
				+ " migrating attributes.");
  }

  Map<String, Object> attributesToMigrate = extractAttributes(session);

  session.invalidate();
  session = request.getSession(true); // we now have a new session

  if (logger.isDebugEnabled()) {
    logger.debug("Started new session: " + session.getId());
  }

  transferAttributes(attributesToMigrate, session);
  return session;
}
```

- `SessionFixationProtectionStrategy`에서 `migrateSession` 설정을 진행하는데,
기존에 session에 있던 정보를 새로운 session에 copy하여 새롭게 만들어서 사용한다.

### changeSession
```java
http.sessionManagement().sessionFixation().changeSessionId();
```

```java
public final class ChangeSessionIdAuthenticationStrategy
		extends AbstractSessionFixationProtectionStrategy {
    
  @Override
  HttpSession applySessionFixation(HttpServletRequest request) {
    request.changeSessionId();
    return request.getSession();
  }
}
```
- changeSession을 적용하면, session에 id를 새롭게 변경 한다.

### none
```java
http.sessionManagement().sessionFixation().none();
```

```java
public final class NullAuthenticatedSessionStrategy implements
		SessionAuthenticationStrategy {

  public void onAuthentication(Authentication authentication,
		HttpServletRequest request, HttpServletResponse response) {
  }
}
```
- none으로 하면 아무런 작업도 하지 않게 된다.

### newSession
```java
http.sessionManagement().sessionFixation().newSession();
```

- `migrateSession`과 동일 하지만, `migrateSessionAttributes`을 `false`로 설정하여, 이전에 Session 값을
copy하지 않고 정말 새롭게 session을 만들어서 사용 한다.

## 유효하지 않는 session을 redirect 시킬 URL 설정
```java
http.sessionManagement().invalidSessionUrl("/");
```
- session이 유효하지 않는 경우 redirect 시킬 url을 설정할 수 있다.
- `InvalidSessionStrategy`에 기본 값으로 생성 되는 `SimpleRedirectInvalidSessionStrategy`에 의해서 redirect가
발생 한다.

```java
InvalidSessionStrategy getInvalidSessionStrategy() {
  if (this.invalidSessionStrategy != null) {
    return this.invalidSessionStrategy;
  }

  if (this.invalidSessionUrl == null) {
    return null;
  }

  this.invalidSessionStrategy = new SimpleRedirectInvalidSessionStrategy(
    this.invalidSessionUrl);
  return this.invalidSessionStrategy;
}
```
- `SessionManagementConfigurer.init()`시에 `InvalidSessionStrategy`가 등록이 된다.


## 동시성 제어 : maximumSession
```java
http.sessionManagement().maximumSessions(1).expiredUrl("/").maxSessionsPreventsLogin(true);
```

- 동시에 인증된 사용자에 대해서 최대 session 갯수를 제한 한다.

## sessionRegistry
```java
public List<SessionInformation> getAllSessions(Object principal,
			boolean includeExpiredSessions) {
  final Set<String> sessionsUsedByPrincipal = principals.get(principal);

  if (sessionsUsedByPrincipal == null) {
    return Collections.emptyList();
  }

  List<SessionInformation> list = new ArrayList<>(
    sessionsUsedByPrincipal.size());

  for (String sessionId : sessionsUsedByPrincipal) {
    SessionInformation sessionInformation = getSessionInformation(sessionId);

    if (sessionInformation == null) {
      continue;
    }

    if (includeExpiredSessions || !sessionInformation.isExpired()) {
      list.add(sessionInformation);
    }
  }

  return list;
}
```
- `SessionRegistry`는 해당 Server에 존재하는 Session에 principal key로 SessionInformations 저장을 하고 있어
해당 user의 session의 갯수를 구하는데 사용할 수 있다. 
- `SessionRegistryImpl`가 기본 구현체이며, `getAllSessions`에서 user에 대한 session 갯수를 가져와서
동시성을 제어하게 된다.

## expire session
- `maximumSessions`를 1로 셋팅 후 session을 공유하지 않는 브라우저에서 각 각 로그인을 하게 되면,
먼저 인증을 한 브라우저에 session이 만료가 된다.
- 만료가 되었을 때, `SessionInformationExpiredStrategy`에 따라서 처리가 되는데 기본값은 `SimpleRedirectSessionInformationExpiredStrategy`이다.
- 따라서 expire에 대해서 redirectUrl을 설정할 수 있다.
 
## preventLogin
- 동시성 제어를 위해서 `maximumSessions`에 대해서 갯수를 제한 했는데, 누군가 로그인을 하면 이전 로그인 사용자의 session이 만료가 된다.
- 이러한 부분을 막을 수 있는 설정이 `maxSessionsPreventsLogin` 설정 인데 maxSession에 도달했을 때
로그인을 못하게 막는 설정이다.
- 기본값은 false로 되어 있어서 maxSession에 도달하게 되어도 로그인이 가능하게 되어 있다.

## session 생성 전략
```java
public enum SessionCreationPolicy {
  /** Always create an {@link HttpSession} */
  ALWAYS,
  /**
  * Spring Security will never create an {@link HttpSession}, but will use the
  * {@link HttpSession} if it already exists
  */
  NEVER,
  /** Spring Security will only create an {@link HttpSession} if required */
  IF_REQUIRED,
  /**
  * Spring Security will never create an {@link HttpSession} and it will never use it
  * to obtain the {@link SecurityContext}
  */
  STATELESS
}
```
- Session을 생성하는 전략은 총 4가지가 존재한다.
### IF_REQUIRED
- Spring Security에 기본 전략이며, 필요시에 session을 생성 한다.
### ALWAYS
- Session을 항상 만든다.
### NEVER
- Spring Security에서 session을 사용하지는 않지만, request에 session이 존재한다면, 사용하게 된다.
### STATELESS
- session을 사용하지 않는 전략 방식이다.