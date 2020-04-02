---
title: "[Spring] Spring Boot, Kotlin 환경에서 Aspect 를 알아보자" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-04-02 23:00:00+09:00 
excerpt: Spring Boot, Kotlin 환경에서 Aspect 기초에 대해서 알아보자.
---

## 들어가며
Aspect에 대해서 적용해보려고 하는데 어떻게 테스트를 해야 하는가에 막혀서 공부한 부분을 작성하려 한다.

Kotlin도 공부하고 싶어서 Kotlin으로 작성을 해보았는데, 아직 익숙하지 않아서 찾아보는 시간이 많이 걸렸다..ㅠㅠ

Spring Boot에서 Aspect을 적용하고 있는데, annotation base aspect 적용 방식과
aspectj에서 제공해주는 annotation 종류와 사용법에 대해서 알아본다.

- Aspect Setting
- Aspect 동작 순서
- Aspect Test

## Aspect Setting
    
```kotlin
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class LogExecutionTime 
```

annotation을 interface를 생성해야 하는데 kotlin에서는 java와 다르게 `annotation class`를 붙여줘야 한다.
- @Target : 해당 어노테이션을 어디에 적용할지를 결정한다.
- @Retention : 해당 어노테이션이 언제까지 유효한지를 정해주는 옵션입니다.
    - SOURCE : compiler 에 의해서 사라지는 어노테이션이며 주석 느낌으로 이해 하면 된다.
    - CLASS : compiler 에 의해서 class 파일에 저장은 되지만 jvm에 실행될 때는 존재하지 않게 한다.
    - RUNTIME : jvm에 의해 class파일이 실행될 시점에도 유효하도록 한다.
- 반드시 `@Target`과 `@Retention`은 붙여줘야 한다.


```kotlin
@Aspect
@Component
class ExampleAspect {
    private val log = LoggerFactory.getLogger(javaClass)

    @Around("@annotation(LogExecutionTime)")
    @Throws(Throwable::class)
    fun logExecutionTime(joinPoint: ProceedingJoinPoint): Any? {
        val start = System.currentTimeMillis()

        var proceed = joinPoint.proceed()
        val executionTime = System.currentTimeMillis() - start

        log.info("${joinPoint.signature} excuted in ${executionTime}ms")
        return proceed
    }
}
```

Aspect에서 제공해주는 `@Aspect` 어노테이션과 bean으로 만들기 위해 `@Compoent` 추가하여 `ExmapleAspect`를 생성한다.

해당 class의 역할은 메소드가 실행될 때 걸리는 시간을 측정하는 간단한 메소드 이다.
- `@Around("@annotation(LogExecutionTime)")` 처럼 annotation base일 경우
 해당 방식으로 annotation class name을 적어주면 된다.

### Kotlin tip!?
- kotlin으로 하다보니 lombok에서 제공해주는 `@Slf4j` 어노테이션이 동작하지 않는다..ㅠbㅠ. 따라서 `LoggerFactory` 통해서 생성해야 한다.
- kotlin에서는 `Object`를 `Any`로 사용해야 하며 null을 허용하기 위해서는 뒤에 `?`를 추가해야 한다.
- kotlin에서는 문자열 템플릿을 제공하여 `${}`를 이용해 String에 값을 바로 표현할 수 있다.  
  
```kotlin
@Service
class ExampleService {
    private val log = LoggerFactory.getLogger(javaClass)

    @LogExecutionTime
    fun serve(flag: Boolean): String {
        log.info("serve")
        if (flag) {
            throw RuntimeException()
        }
        return "serve";
    }
}
```

aspect까지 생성 했으니 이제 적용할 `Service`를 생성한다.

boolean 파라메터를 받고 log 찍고 String을 리턴하는 간단한 메소드이다.

```kotlin
@RestController
class ExampleController(@Autowired private val exampleService: ExampleService) {
    @GetMapping("/around")
    fun logging(@RequestParam(defaultValue = "false") flag: Boolean): Map<String, String> {

        return mapOf("key" to exampleService.serve(flag))
    }
}
```

`Service`까지만 적용하고 aspect를 테스트해보려고 하였으나, 테스트가 잘 안되어 `spring web`까지 추가하였다.

왜 안되었는지는 다음에 알아보려고 한다.

## Aspect 동작 순서
Aspectj 에서 제공해주는 annotation 중에서 알아볼 것은 5개 이다.
- @Before
- @Around
- @After
- @AfterReturning
- @AfterThrowing

동작 순서는 리스트에 작성 순서대로 동작하나 좀 더 자세하게 적자면 아래와 같다.

1. Before
2. Around에 joinPoint.proceed() 전
3. 해당 method 실행
4. Around에 joinPoint.proceed() 후
5. After
6. AfterRetuning or AfterThrowing

```kotlin
@Aspect
@Component
class ExampleAspect {
    private val log = LoggerFactory.getLogger(javaClass)

    @Before("@annotation(LogExecutionTime)")
    fun before(joinPoint: JoinPoint) {
        log.info("before")
    }

    @Around("@annotation(LogExecutionTime)")
    @Throws(Throwable::class)
    fun logExecutionTime(joinPoint: ProceedingJoinPoint): Any? {
        val start = System.currentTimeMillis()

        // 파라메터에 값을 전달하기 위해 Any array를 생성해서 proceed의 파라메터로 전달 
//        val flag = true
//        var proceed = joinPoint.proceed(arrayOf<Any>(flag))

        var proceed = joinPoint.proceed()
        val executionTime = System.currentTimeMillis() - start

        log.info("${joinPoint.signature} excuted in ${executionTime}ms")
        return proceed
    }

    @After("@annotation(LogExecutionTime)")
    fun after(joinPoint: JoinPoint) {
        log.info("after")
    }

    @AfterReturning(value = "@annotation(LogExecutionTime)", returning = "resource")
    fun afterReturning(resource: Any?) {
        log.info("after Returning $resource")
    }

    @AfterThrowing(value = "@annotation(LogExecutionTime)", throwing = "exception")
    fun afterThrowing(exception: Exception) {
        log.info("after Throwing", exception)
    }

}
```

- `@Around`의 경우는 파라메터 값을 주입이 가능하며, `Object Array`로 전달하면 파라메터의 값을 변경할 수 있다.
- `@AfterRetuning`의 경우는 해당 메소드에서 return 값을 전달 받아서 작업을 진행할 수 있게 해준다.
- `@AfterThrowing`의 경우는 해당 메소드가 Exception이 발생하는 경우 실행된다.
 
## Aspect Test
Aspect 테스트를 어떻게 진행해야 하나 고민하다가 web 디펜던시도 추가하였으니 MockMvc를 이용해서 테스트 해보았다.

```kotlin
@AutoConfigureMockMvc(printOnlyOnFailure = false)
@SpringBootTest
internal class ExampleControllerTest(@Autowired private val mockMvc: MockMvc) {
    val log = LogFactory.getLog(javaClass)
    val objectMapper = jacksonObjectMapper()

    @Test
    internal fun aspectTest() {
        val result = mockMvc.get("/around")
                .andDo { log() }
                .andExpect { status { is2xxSuccessful } }
                .andReturn()

        val readValue = objectMapper.readValue<Map<String, String>>(result.response.contentAsString)


        log.info("result $readValue")
    }
}
``` 

### `@AutoConfigureMockMvc(printOnlyOnFailure = false)`
- 해당 어노테이션을 추가하여 MockMvc를 Autowired로 injection할 수 있게 한다.
- `printOnlyOnFailure`를 false하여 성공했을 때도 로그를 찍도록 하여 `.andDo { log() }`가 없더라도 로그가 출력될 수 있게 한다.

### @SpringBootTest
- `@MockMvcTest`를 사용하는 경우 `@Service`레이어가 포함되지 않아 `@SpringBootTest`를 사용한다.

### jacksonObjectMapper
- kotlin에서는 objectMapper를 사용하기 위해서는 코틀린 전용 `jackson-module-kotlin` 디펜던시를 추가해야 한다.
- 해당 디펜던시를 추가하면 `jacksonObjectMapper`를 사용할 수 있게 된다. 

## 마치며
Aspect를 간단하게 테스트 시작했는데 생각보다 kotlin에 대해서 더 많이 공부하게 된거 같다.

list, map도 만들 줄 몰랐고 java에서는 잘 동작했는데 왜 안되는지 하나씩 검색하면서 하다보니
시간이 더 많이 걸렸던 것 같다.

그래도 공부하고 싶은 Kotlin은 포기하고 싶지 않아서 다음 sample project를 할때도 Kotlin으로 작성할 예정이다.

- - - 
[aspect](https://github.com/KangWooJin/spring-study/tree/master/aspect)
관련 example code는 github에 올려두었으니 참고~!
 