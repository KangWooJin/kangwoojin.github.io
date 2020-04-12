---
title: "[Spring] RestTemplate Error Handling 하기" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-04-12 20:00:00+09:00 
excerpt: Spring에서 client로 많이 사용하는 RestTemplate에 error handling 해보자
---

## 들어가며
Spring에서 다른 서버에 api를 호출하기 위해 자주 사용되는 `RestTemplate`에 error가 발생했을 때
원하는 동작으로 error handling을 하는 방법에 대해서 알아본다.

## Setting

간단하게 테스트를 할 예정이라 해당 dependency만 추가하고 진행 한다.

- lombok
- spring boot web

MockServer를 이용해도 되지만 local에서 간단하게 서버를 띄워두고 테스트를 진행할 예정이라 간단한 controller를 추가한다.

## ExampleController
```kotlin
@RestController
class ExampleController {

    @GetMapping("/example")
    fun example(): String {
        return "success"
    }

    @PostMapping("/badRequest")
    fun badRequest(): ResponseEntity<ExampleResponse> {
        val response = ExampleResponse("message", "1", UUID.randomUUID())
        return ResponseEntity(response, HttpStatus.BAD_REQUEST)
    }

    class ExampleResponse(var message: String?, var version: String?, var uuid: UUID?)
}
```

- `http://localhost:8080/example`로 요청이 들어오는 경우 `success`로 response를 내려준다.
- `http://localhost:8080/badRequest`로 요청이 들어오는 경우 `StatusCode`를 400에 원하는 response를 내려 준다.


## ExampleRestTemplate

```kotlin
@Component
class ExampleRestTemplate {
    private val restTemplate: RestTemplate = RestTemplate()
    private val log = LoggerFactory.getLogger(javaClass)
    private var errorHandlerRestTemplate: RestTemplate? = null

    constructor(requestBuilder: RestTemplateBuilder) {
        errorHandlerRestTemplate = requestBuilder.errorHandler(RestTemplateResponseErrorHandler()).build()
    }

    fun successExample(): String {
        val result = restTemplate.getForEntity<String>("http://localhost:8080/example")
        print(result)

        return result.body.toString()
    }

    @Throws(BadRequest::class)
    fun failExample(): String? {
        var result: ResponseEntity<String>? = null
        try {
            result = restTemplate.getForEntity<String>("http://localhost:8080/badRequest")
        } catch (e: HttpClientErrorException) {
            log.error("exception {}", e.message, e)
            throw BadRequest(e)
        }
        print(result)

        return result.body?.toString()
    }

    fun failExampleByErrorHandler(): ExampleController.ExampleResponse? {
        val result = errorHandlerRestTemplate?.postForEntity<ExampleController.ExampleResponse>("http://localhost:8080/badRequest")

        printResponse(result)

        return result?.body
    }


    private fun <T> printResponse(result: ResponseEntity<T>?) {
        if (result == null) {
            return
        }
        log.info("result status code {}", result.statusCode)
        log.info("result body {}", result.body)
        log.info("result headers {}", result.headers)
        log.info("result statusCodeValue {}", result.statusCodeValue)
    }
}
```

- `RestTemplate` 정상 동작하고 있는지 확인 하기 위해 `/example`을 호출하는 테스트를 작성 한다.

```kotlin
@SpringBootTest
internal class ExampleRestTemplateTest(@Autowired private val exampleRestTemplate: ExampleRestTemplate) {
    private val log = LoggerFactory.getLogger(javaClass)
    
    @Test
    internal fun exampleTest() {
        val result = exampleRestTemplate.successExample();

        assertThat(result).isEqualTo("success")
    }
}
```

- 해당 테스트를 성공 시키기 위해 `successExample` 메소드를 구현한다.

```kotlin
@Component
class ExampleRestTemplate {
    private val restTemplate: RestTemplate = RestTemplate()
    private val log = LoggerFactory.getLogger(javaClass)

    fun successExample(): String {
        val result = restTemplate.getForEntity<String>("http://localhost:8080/example")
        print(result)

        return result.body.toString()
    }
}
```

- 성공 케이스가 아닌 실패 케이스가 발생했을 때 어떻게 동작하는지 확인

```kotlin
@Test
internal fun badRequestTest() {
    assertThrows<BadRequest> {
        exampleRestTemplate.failExample()
    }
}
```

- 해당 테스트를 성공 시키기 위해서 `/badRequest`를 호출하는 failExample을 구현
```kotlin
    @Throws(BadRequest::class)
    fun failExample(): String? {
        var result: ResponseEntity<String>? = null
        try {
            result = restTemplate.getForEntity<String>("http://localhost:8080/badRequest")
        } catch (e: HttpClientErrorException) {
            log.error("exception {}", e.message, e)
            throw BadRequest(e)
        }
        print(result)

        return result.body?.toString()
    }
```

- 기본적인 RestTemplate의 경우 2xx의 케이스가 아닌 경우 에러로 판단하고 다음 라인이 실행되지 않음
- 따라서 `HttpClientErrorException`에 `e.getStatusCode()`를 통해서 error를 판단해야 한다.
- 매번 try catch를 통해서 할 순 없으니 `RestTemplate`에 `errorHandle`을 추가하여 관리한다.

```kotlin
class RestTemplateResponseErrorHandler : DefaultResponseErrorHandler() {
    @Throws(IOException::class)
    override fun hasError(httpResponse: ClientHttpResponse): Boolean {
        return super.hasError(httpResponse)
    }

    @Throws(IOException::class)
    override fun handleError(httpResponse: ClientHttpResponse) {
        if (httpResponse.statusCode.series() === HttpStatus.Series.SERVER_ERROR) {
            throw RuntimeException()
        } else if (httpResponse.statusCode.series() === HttpStatus.Series.CLIENT_ERROR) {
            // handle CLIENT_ERROR
            if (httpResponse.statusCode === HttpStatus.NOT_FOUND) {
                throw NotFoundException(message = httpResponse.body.toString(), exception = null)
            }
        }
    }
}
```

```kotlin
private var errorHandlerRestTemplate: RestTemplate? = null

constructor(requestBuilder: RestTemplateBuilder) {
    errorHandlerRestTemplate = requestBuilder.errorHandler(RestTemplateResponseErrorHandler()).build()
}
```

- `RestTemplateBuilder`를 이용해 `RestTemplate`에 `errorHandler`를 추가 해준다.
- `errorHandler`는 `ResponseErrorHandler`를 상속받아서 구현해야 하며 `DefaultResponseErrorHandler`가 이미 존재하고 있다.
- client error(4xx)와 server error(5xx)의 경우 error로 판단하고 handleError에서 처리해준다.
- server error의 경우는 딱히 처리해줄 것이 없어 `RuntimeException`으로 하고 `Client Error`에 대해서 throw를 하지 않으면
문제 없이 다음 행으로 진행 가능하다.
 

```kotlin
@Test
internal fun errorHandlerTest() {
    val actual = exampleRestTemplate.failExampleByErrorHandler()
    assertThat(actual).isNotNull()
}
```

- error가 발생하기에 다음 행 진행이 안되었는데 error handler를 추가하여 4xx일때 무시하도록 셋팅
- status code가 4xx 지만 try catch를 하지 않고도 다음 행으로 진행할 수 있다.  


## 마치며
RestTemplate에 대해서 매번 간단하게만 사용하고 있어서 커스텀할 수 있는 부분에 대해서 모르고 있었는데
간단하게 나마 공부해보고 정리해보았다.

나중에는 RestTemplate이 아니라 넷플릭스에서 오픈소스로 공개한 feign 라이브러리를 이용해서 개발을 하게 될거라 생각한다.

따라서 다음엔 feign에 대해서 공부해보려고 한다~!

- - - 
[RestTemplateExample](https://github.com/KangWooJin/spring-study/tree/master/rest-template)
관련 example code는 github에 올려두었으니 참고~!