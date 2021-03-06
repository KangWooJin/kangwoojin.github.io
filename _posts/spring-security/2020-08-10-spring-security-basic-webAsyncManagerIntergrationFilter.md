---
title: "[Spring Security] Spring Security Basic - WebAsyncManagerIntegrationFilter" 
categories:
  - Programing
tags:
  - SpringSecurity
toc: true
toc_sticky: true
date: 2020-08-10 23:00:00+09:00
last_modified_at: 2020-08-11 23:00:00  
excerpt: Callable을 이용해 Async 방식의 response를 전달할때 SecurityContextHolder에 값이 어떻게 채워지는지 알아보자.
---

## 들어가며
Callable을 이용해 Async 방식의 response를 전달할때 SecurityContextHolder에 값이 어떻게 채워지는지 알아보자.

## WebAsyncManagerIntegrationFilter

- `WebAsyncManagerIntegrationFilter`를 등록하면 `SecurityContextHolder`의 값이 Async 방식의 response일때도 값이 채워지게 된다.
- 그 이유에 대해서 알아 보자.

```java
public final class WebAsyncManagerIntegrationFilter extends OncePerRequestFilter {
	private static final Object CALLABLE_INTERCEPTOR_KEY = new Object();

	@Override
	protected void doFilterInternal(HttpServletRequest request,
			HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {
		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		SecurityContextCallableProcessingInterceptor securityProcessingInterceptor = (SecurityContextCallableProcessingInterceptor) asyncManager
				.getCallableInterceptor(CALLABLE_INTERCEPTOR_KEY);
		if (securityProcessingInterceptor == null) {
			asyncManager.registerCallableInterceptor(CALLABLE_INTERCEPTOR_KEY,
					new SecurityContextCallableProcessingInterceptor());
		}

		filterChain.doFilter(request, response);
	}
}
```


- `WebAsyncManagerIntegrationFilter`의 코드는 정말 간단 하게 되어 있다.. 여기만 보고는 이해하기 힘들다..
- `SecurityFilterChains`에서 첫번째로 등록되어 있는 Filter이며, `WebAsyncManagerIntegrationFilter` 역할은
 `WebAsyncManager`에 `SecurityContextCallableProcessingInterceptor`를 등록하는 Filter이다.

### SecurityContextCallableProcessingInterceptor

```java
public final class SecurityContextCallableProcessingInterceptor extends
		CallableProcessingInterceptorAdapter {
	private volatile SecurityContext securityContext;

	@Override
	public <T> void beforeConcurrentHandling(NativeWebRequest request, Callable<T> task) {
		if (securityContext == null) {
			setSecurityContext(SecurityContextHolder.getContext());
		}
	}

	@Override
	public <T> void preProcess(NativeWebRequest request, Callable<T> task) {
		SecurityContextHolder.setContext(securityContext);
	}

	@Override
	public <T> void postProcess(NativeWebRequest request, Callable<T> task,
			Object concurrentResult) {
		SecurityContextHolder.clearContext();
	}
}
```

- `SecurityContextCallableProcessingInterceptor`는 현재 Thread에 있는 `SecurityContext`를 갖고 있다가, set, clear해주는 역할이다.
- `beforeConcurrentHandling`에서 현재 Thread에 갖고 있는 `SecurityContext`를 주입해 준다.
    - 여기서 주입을 해주는 이유는 Async로 동작할 때 다른 Thread로 넘어가기에 ThreadLocal의 값이 바뀌게 되니, 현재 Thread의 
    `SecurityContext`의 값을 저장해두는 것이다.    
- `SecurityContextCallableProcessingInterceptor`에 보면, `SecurityContextHolder.setContext`를 통해서 
현재 Thread에 `SecurityContext`등록한다.
- `SecurityContextHolder.clearContext`에서는 현재 Thread에 `SecurityContext`를 clear하는 역할을 한다.
- `SecurityContextCallableProcessingInterceptor`를 통해서 `SecurityContextHolder`에 값을 채우는 것은 확인하였으니,
어디서 호출이 되는지 알아보자.

### CallableMethodReturnValueHandler
- 더 자세하게 들어가면 handler를 선택하는 부분으로 넘어가게 되는데, 그 부분은 스킵한다.
- 선택된 handler는 `CallableMethodReturnValueHandler`이고, 해당 handler를 통해서 request를 처리하게 된다.

```java
public class CallableMethodReturnValueHandler implements HandlerMethodReturnValueHandler {

	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return Callable.class.isAssignableFrom(returnType.getParameterType());
	}

	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
		if (returnValue == null) {
			mavContainer.setRequestHandled(true);
			return;
		}
		Callable<?> callable = (Callable<?>) returnValue;
		WebAsyncUtils.getAsyncManager(webRequest).startCallableProcessing(callable, mavContainer);
	}
}
``` 

- `CallableMethodReturnValueHandler`에서 `WebAsyncManager`를 가져오고, `startCallableProcessing`를 통해서
Callable을 처리하게 된다.


```java
public void startCallableProcessing(final WebAsyncTask<?> webAsyncTask, Object... processingContext)
			throws Exception {
    ...
  interceptorChain.applyBeforeConcurrentHandling(this.asyncWebRequest, callable);
  startAsyncProcessing(processingContext);
  try {
    Future<?> future = this.taskExecutor.submit(() -> {
      Object result = null;
      try {
        interceptorChain.applyPreProcess(this.asyncWebRequest, callable);
        result = callable.call();
      }
      catch (Throwable ex) {
        result = ex;
      }
      finally {
        result = interceptorChain.applyPostProcess(this.asyncWebRequest, callable, result);
      }
      setConcurrentResultAndDispatch(result);
    });
    interceptorChain.setTaskFuture(future);
  }
  catch (RejectedExecutionException ex) {
    Object result = interceptorChain.applyPostProcess(this.asyncWebRequest, callable, ex);
    setConcurrentResultAndDispatch(result);
    throw ex;
  }
}
```
 
- `startCallableProcessing`에서 처리하는 역할은 많지만, 확인 하고 싶은 부분은 `SecurityContextHolder`에 값을 언제 채우는지만 알아 본다. 
- `applyBeforeConcurrentHandling`을 통해서 현재 Thread가 갖고 있는 `SecurityContext`를 `SecurityContextCallableProcessingInterceptor`
에 저장 해둔다.
- 그 후 `applyPreProcess`에서 `SecurityContext`를 `setContext` 해주고, `callable.call();`을 통해서 callable을 처리 한다.
- 마지막으로 `applyPostProcess`에서 Async의 Thread에 할당된 `SecurityContext`를 clear하는 로직이 동작하게 된다.

## @Async를 이용하는 경우는 어떻게 되나?
- `@Async`를 이용하는 경우 `WebAsyncManager`를 통해서 `SecurityContext`를 주입해주는 방식을 사용할 수 없으니,
`SecurityContextHolder`의 값이 빈 값이 된다. 
- 어떻게 하면, 다른 Thread에도 `SecurityContextHolder` 제공할 수 있는지 확인 해보자.

### SecurityContextHolder strategy
- [SecurityContextHolder]({% post_url spring-security/2020-08-06-spring-security-basic-security-context-holder %})
에서 사용되는 ThreadLocal 전략에 대해서 알아보았다.
- `SecurityContextHolder`의 `strategy`를 `MODE_THREADLOCAL`에서 `MODE_INHERITABLETHREADLOCAL`로 변경하면 된다.
- 한번 설정하게 되면, application이 떠있는 동안 계속 적용되기에 Application load시에 적용해주면 된다.


```java
public static void setStrategyName(String strategyName) {
    SecurityContextHolder.strategyName = strategyName;
    initialize();
}
```

- `SecurityContextHolder`에 `setStrategyName`를 통해서 strategy를 변경하게 되면,
`initialize()`를 통해서 ThreadLocal의 종류가 바뀌게 된다.

```java
private static void initialize() {
  if (!StringUtils.hasText(strategyName)) {
    // Set default
		strategyName = MODE_THREADLOCAL;
  }

  if (strategyName.equals(MODE_THREADLOCAL)) {
    strategy = new ThreadLocalSecurityContextHolderStrategy();
  }
  else if (strategyName.equals(MODE_INHERITABLETHREADLOCAL)) {
    strategy = new InheritableThreadLocalSecurityContextHolderStrategy();
  }
  else if (strategyName.equals(MODE_GLOBAL)) {
    strategy = new GlobalSecurityContextHolderStrategy();
  }
  else {
    // Try to load a custom strategy
		try {
		  Class<?> clazz = Class.forName(strategyName);
		  Constructor<?> customStrategy = clazz.getConstructor();
		  strategy = (SecurityContextHolderStrategy) customStrategy.newInstance();
		}
		catch (Exception ex) {
		  ReflectionUtils.handleReflectionException(ex);
		}
  }
  initializeCount++;
}
```

- 종류는 총 3가지가 존재하며, default는 `MODE_THREADLOCAL`로 사용 된다.
- `MODE_GLOBAL`의 경우는 전체 Thread에 공통된 값을 가지게 설정하게 된다.


## 마치며
- Callable request가 요청왔을 때 SecurityContextHolder에 값이 채워지고 사라지는 호출 순서는 아래와 같다.

>1. Callable Request 요청
>2. `WebAsyncManagerIntegrationFilter`에서 `WebAsyncManager`에 `SecurityContextCallableProcessingInterceptor` 등록
>3. Handler 처리기에서 `CallableMethodReturnValueHandler`가 선택되어 request 처리
>4. `applyBeforeConcurrentHandling`를 통해서 Async Thread로 넘어가기 전 현재 Thread에 있는 `SecurityContext`를 
 `SecurityContextCallableProcessingInterceptor`에 저장
>5. `applyPreProcess`에서 Async Thread에 SecurityContext set
>6. `applyPostProcess`에서 Async Thread에 SecurityContext clear

- `@Async`를 사용할 때, 즉 currentThread에서 다른 Thread로 넘어가게 되면 `SecurityContextHolder`의 값을
공유하지 못 하게 된다.
- 공유하기 위해서는 `SecurityContextHolder`의 ThreadLocal strategy를 `MODE_INHERITABLETHREADLOCAL`로 변경 하면 해결할 수 있다.
