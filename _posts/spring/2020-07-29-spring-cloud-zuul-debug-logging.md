---
title: "[Spring] Spring Cloud Zuul debug logging 설정하기" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-07-29 08:00:00+09:00 
excerpt: Spring Cloud Zuul에서 debug logging을 설정 해보자
---

## 들어가며
spring cloud zuul에 대해서 설명은 포함되어 있지 않아, 기본지식이 없는 경우 이해하기 어려울 수 있다.

zuul을 사용해서 request, response로 전달되고 있는 데이터가 궁금하여, zuul에 있는 request, response를
그대로 출력해서 확인하고 있었는데, 팀원분이 해당 부분을 전체적으로 적용할 수 있는 방법을 알려주어 찾아보게 되었다.

debug header를 추가해보았는데, debug header를 추가했을 때 header size 초과가 발생할 수 있어서,
좋아보이지 않았고 바로 출력되는 것이 아니라서 보기가 힘들었다.

그러다가 공식 문서를 읽다가 sample app에서 하는 방식을 확인해보니, 직접적으로 제공해주는 것이 없고
직접 구현해서 사용하고 있어서 필요시에 추가하는 방향으로 개발을 하게 되었다.


## DebugFilter 활성화

- `DebugFilter`는 zuul에서 제공해주는 `RequestContext`에 debug를 활성화 해주는 `ZuulFilter`이다.
- 해당 부분을 활성화 하기 위해서는 여러 방법이 존재할 수 있지만, 간단하게 적용하는 방법을 소개 한다.

### property 방식 

```yaml
zuul.debug.request: true
```

- IDE에 자동완성이 되지 않아 해당 property가 없어보이지만, Zuul에서 `DynamicBooleanProperty`를 이용해서
 `DebugFilter`에 해당 값을 확인하는 로직이 들어가 있다.
- 해당 값이 true인 경우 `DebugFilter`가 실행되게 된다.
- `DebugFilter`에 run() 부분을 보면 debugRouting, debugRequest를 true하는 부분이 포함되어 있다.

```java
public class DebugFilter extends ZuulFilter {

	private static final DynamicBooleanProperty ROUTING_DEBUG = DynamicPropertyFactory
			.getInstance().getBooleanProperty(ZuulConstants.ZUUL_DEBUG_REQUEST, false);

	private static final DynamicStringProperty DEBUG_PARAMETER = DynamicPropertyFactory
			.getInstance().getStringProperty(ZuulConstants.ZUUL_DEBUG_PARAMETER, "debug");

	@Override
	public String filterType() {
		return PRE_TYPE;
	}

	@Override
	public int filterOrder() {
		return DEBUG_FILTER_ORDER;
	}

	@Override
	public boolean shouldFilter() {
		HttpServletRequest request = RequestContext.getCurrentContext().getRequest();
		if ("true".equals(request.getParameter(DEBUG_PARAMETER.get()))) {
			return true;
		}
		return ROUTING_DEBUG.get();
	}

	@Override
	public Object run() {
		RequestContext ctx = RequestContext.getCurrentContext();
		ctx.setDebugRouting(true);
		ctx.setDebugRequest(true);
		return null;
	}

}
```

### RequestContext 방식

```java
RequestContext ctx = RequestContext.getCurrentContext();
ctx.setDebugRouting(true);
ctx.setDebugRequest(true);
```

- `RequestContext`에 `setDebugRouting`, `setDebugRequest`의 값을 직접 true로 변경하게 되면,
`DebugFilter`를 통하지 않고 zuul이 debug 환경으로 설정할 수 있다.
- `RequestContext`가 ThreadLocal 방식과 동일하여, 필요한 request에만 debug 설정을 true로 변경하여 확인이 가능하다.
 
## Debug 데이터 셋팅

- `requestDebug`의 데이터는 zuul 라이브러리에서 제공해주는 부분이 없다, 따라서 직접 구현을 해야 한다.
- sample app에 있는 방식을 보고 java 방식으로 변경하였다.

### Client -> Zuul
```java
@Component
@ConditionalOnProperty(name = "zuul.debug.request", havingValue = "true")
public class DebugRequestFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return Integer.MAX_VALUE;
    }

    @Override
    public boolean shouldFilter() {
        return Debug.debugRequest();
    }

    @SneakyThrows
    @Override
    public Object run() throws ZuulException {
        HttpServletRequest req = RequestContext.getCurrentContext().getRequest();

        // client -> zuul
        Debug.addRequestDebug("REQUEST:: Client > " + req.getScheme() + ' ' + req.getRemoteAddr() + ':' + req.getRemotePort());
        Debug.addRequestDebug("REQUEST:: Request > " + req.getMethod() + ' ' + req.getRequestURI() + ' ' + req.getProtocol());

        Iterator headerIt = req.getHeaderNames().asIterator();
        while (headerIt.hasNext()) {
            String name = (String) headerIt.next();
            String value = req.getHeader(name);
            Debug.addRequestDebug("REQUEST:: Header > " + name + ':' + value);
        }

        final RequestContext ctx = RequestContext.getCurrentContext();
        if (!StringUtils.equalsIgnoreCase(req.getMethod(), "GET") && !ctx.isChunkedRequestBody()) {
            InputStream inp = ctx.getRequest().getInputStream();
            String body = "";
            if (inp != null) {
                body = new String(inp.readAllBytes(), StandardCharsets.UTF_8);
                Debug.addRequestDebug("REQUEST:: RequestBody > " + body);

            }
        }
        return null;
    }
}
```

### Zuul <-> Origin
```java
@Slf4j
@Component
@ConditionalOnProperty(name = "zuul.debug.request", havingValue = "true")
public class DebugRoutingFilter extends ZuulFilter {
    @Autowired
    private ProxyRequestHelper helper;

    @Override
    public String filterType() {
        return ROUTE_TYPE;
    }

    @Override
    public int filterOrder() {
        return SIMPLE_HOST_ROUTING_FILTER_ORDER + 1;
    }

    @Override
    public boolean shouldFilter() {
        return Debug.debugRequest();
    }

    @SneakyThrows
    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        MultiValueMap<String, String> requestHeaders = helper.buildZuulRequestHeaders(request);
        String uri = helper.buildZuulRequestURI(request);
        String method = request.getMethod();
        String query = Optional.ofNullable(request.getQueryString())
                               .map(it -> '?' + it)
                               .orElse("");
        String requestBody = "";

        String host = Optional.ofNullable(ctx.getRouteHost())
                              .map(URL::getHost)
                              .orElseGet(() -> (String) ctx.get(SERVICE_ID_KEY));

        // zuul -> origin
        Debug.addRequestDebug("ZUUL:: Origin > " + host);
        requestHeaders.forEach((key, value) -> Debug.addRequestDebug("ZUUL:: Header > " + key + ' ' + value));
        Debug.addRequestDebug("ZUUL:: Request > " + method + ' ' + uri + query + ' ' + request.getProtocol());
        if (!StringUtils.equalsIgnoreCase(method, "GET") && !ctx.isChunkedRequestBody()) {
            InputStream inp = ctx.getRequest().getInputStream();
            if (inp != null) {
                requestBody = new String(inp.readAllBytes(), StandardCharsets.UTF_8);
                Debug.addRequestDebug("ZUUL:: RequestBody > " + requestBody);
            }
        }
        Debug.addRequestDebug("ZUUL:: ResponseCode > " + ctx.getResponseStatusCode());

        // origin -> zuul
        ctx.getOriginResponseHeaders().forEach(pair -> {
            if (helper.isIncludedHeader(pair.first())) {
                Debug.addRequestDebug("ORIGIN:: Header < " + pair.first() + ' ' + pair.second());
            }
        });

        return null;
    }
}
```

### Zuul -> Client
```java
@Component
@ConditionalOnProperty(name = "zuul.debug.request", havingValue = "true")
public class DebugSendResponseFilter extends ZuulFilter {
    @Override
    public String filterType() {
        return POST_TYPE;
    }

    @Override
    public int filterOrder() {
        return STATS_FILTER_ORDER - 1;
    }

    @Override
    public boolean shouldFilter() {
        return Debug.debugRequest();
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        List<Pair<String, String>> zuulResponseHeaders = ctx.getZuulResponseHeaders();

        // zuul -> client
        zuulResponseHeaders.forEach(it -> Debug.addRequestDebug("OUTBOUND:: Header <  " + it.first() + ':' + it.second()));

        return null;
    }
}
```

- `routingDebug`는 zuul에서 제공해주는 `FilterProcessor`에서 데이터를 생성 한다.
- `routingDebug`는 zuul에 filter에 동작과 `RequestContext`의 수정과정을 보여주고 있어, `requestDebug`와 목적이 다르다.

```java
public static void compareContextState(String filterName, RequestContext copy) {        RequestContext context = RequestContext.getCurrentContext();
    Iterator<String> it = context.keySet().iterator();
    String key = it.next();
    while (key != null) {
        if ((!key.equals("routingDebug") && !key.equals("requestDebug"))) {
            Object newValue = context.get(key);
            Object oldValue = copy.get(key);
            if (oldValue == null && newValue != null) {
                addRoutingDebug("{" + filterName + "} added " + key + "=" + newValue.toString());
            } else if (oldValue != null && newValue != null) {
                if (!(oldValue.equals(newValue))) {
                    addRoutingDebug("{" +filterName + "} changed " + key + "=" + newValue.toString());
                }
            }
        }
        if (it.hasNext()) {
             key = it.next();
        } else {
             key = null;
        }
    }
}
```

## Debug logging

```java
@Slf4j
@Component
@ConditionalOnProperty(name = "zuul.debug.request", havingValue = "true")
public class StatsFilter extends ZuulFilter {
    public static final int STATS_FILTER_ORDER = 2000;

    @Override
    public String filterType() {
        return POST_TYPE;
    }

    @Override
    public int filterOrder() {
        return STATS_FILTER_ORDER;
    }

    @Override
    public boolean shouldFilter() {
        return Debug.debugRequest();
    }

    @Override
    public Object run() {
        dumpRoutingDebug();
        dumpRequestDebug();

        return null;
    }

    public void dumpRequestDebug() {
        List<String> rd = (List<String>) RequestContext.getCurrentContext().get("requestDebug");
        rd.forEach(it -> log.debug("REQUEST_DEBUG::{}", it));
    }

    public void dumpRoutingDebug() {
        List<String> rd = (List<String>) RequestContext.getCurrentContext().get("routingDebug");
        rd.forEach(it -> log.debug("ZUUL_DEBUG::{}", it));
    }
}
```

- `requestDebug`, `routingDebug`에 설정된 값을 logger를 통해서 print 해주는 Filter를 추가해야 한다.
- 해당 부분도 zuul에서 제공해주지 않는다..ㅠ
- 해당 debug filter를 적용하면 아래와 비슷한 데이터가 출력 된다.

```
ZUUL_DEBUG::Filter pre 5 PreDecorationFilter
ZUUL_DEBUG::Filter {PreDecorationFilter TYPE:pre ORDER:5} Execution time = 12ms
ZUUL_DEBUG::{PreDecorationFilter} added originResponseHeaders=[com.netflix.util.Pair@64edca63]
ZUUL_DEBUG::{PreDecorationFilter} added routeHost=http://apache.org/
ZUUL_DEBUG::Filter pre 10000 DebugRequest
ZUUL_DEBUG::Filter {DebugRequest TYPE:pre ORDER:10000} Execution time = 36ms
ZUUL_DEBUG::Invoking {route} type filters
ZUUL_DEBUG::Filter route 100 SimpleHostRoutingFilter
ZUUL_DEBUG::Filter {SimpleHostRoutingFilter TYPE:route ORDER:100} Execution time = 251ms
ZUUL_DEBUG::{SimpleHostRoutingFilter} added responseStatusCode=304
ZUUL_DEBUG::{SimpleHostRoutingFilter} added hostZuulResponse=HTTP/1.1 304 Not Modified [Date: Sun, 09 Jun 2013 02:09:19 GMT, Server: Apache/2.4.4 (Unix) mod_wsgi/3.4 Python/2.7.3 OpenSSL/1.0.0g, Connection: Keep-Alive, Keep-Alive: timeout=5, max=100, ETag: "90a0-4deae5495ba47", Expires: Sun, 09 Jun 2013 03:09:19 GMT, Cache-Control: max-age=3600]
ZUUL_DEBUG::{SimpleHostRoutingFilter} added zuulResponseHeaders=[com.netflix.util.Pair@28aa2b8a, com.netflix.util.Pair@efa5136, com.netflix.util.Pair@1615a33c, com.netflix.util.Pair@c8b48f6b, com.netflix.util.Pair@2bbdebb3]
ZUUL_DEBUG::{SimpleHostRoutingFilter} added zuulRequestHeaders={}
ZUUL_DEBUG::{SimpleHostRoutingFilter} added responseGZipped=false
ZUUL_DEBUG::Invoking {post} type filters
ZUUL_DEBUG::Filter post 1000 SendResponseFilter
ZUUL_DEBUG::Filter {SendResponseFilter TYPE:post ORDER:1000} Execution time = 22ms
ZUUL_DEBUG::Filter post 2000 Stats
```

```
REQUEST_DEBUG::REQUEST:: http 0:0:0:0:0:0:0:1:51254
REQUEST_DEBUG::REQUEST:: > GET / HTTP/1.1
REQUEST_DEBUG::REQUEST:: > Host:localhost:8080
REQUEST_DEBUG::REQUEST:: > Connection:keep-alive
REQUEST_DEBUG::REQUEST:: > Cache-Control:max-age=0
REQUEST_DEBUG::REQUEST:: > Accept:text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
REQUEST_DEBUG::REQUEST:: > User-Agent:Mozilla/5.0 (Macintosh; Intel Mac OS X 10_7_5) AppleWebKit/537.31 (KHTML, like Gecko) Chrome/26.0.1410.65 Safari/537.31
REQUEST_DEBUG::REQUEST:: > DNT:1
REQUEST_DEBUG::REQUEST:: > Accept-Encoding:gzip,deflate,sdch
REQUEST_DEBUG::REQUEST:: > Accept-Language:en-US,en;q=0.8
REQUEST_DEBUG::REQUEST:: > Accept-Charset:ISO-8859-1,utf-8;q=0.7,*;q=0.3
REQUEST_DEBUG::REQUEST:: > If-None-Match:"90a0-4deae5495ba47"
REQUEST_DEBUG::REQUEST:: > If-Modified-Since:Sun, 09 Jun 2013 01:10:31 GMT
REQUEST_DEBUG::REQUEST:: > 
REQUEST_DEBUG::ZUUL:: host=http://apache.org/
REQUEST_DEBUG::ZUUL::> Host  localhost:8080
REQUEST_DEBUG::ZUUL::> Connection  keep-alive
REQUEST_DEBUG::ZUUL::> Cache-Control  max-age=0
REQUEST_DEBUG::ZUUL::> Accept  text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
REQUEST_DEBUG::ZUUL::> User-Agent  Mozilla/5.0 (Macintosh; Intel Mac OS X 10_7_5) AppleWebKit/537.31 (KHTML, like Gecko) Chrome/26.0.1410.65 Safari/537.31
REQUEST_DEBUG::ZUUL::> DNT  1
REQUEST_DEBUG::ZUUL::> Accept-Language  en-US,en;q=0.8
REQUEST_DEBUG::ZUUL::> Accept-Charset  ISO-8859-1,utf-8;q=0.7,*;q=0.3
REQUEST_DEBUG::ZUUL::> If-None-Match  "90a0-4deae5495ba47"
REQUEST_DEBUG::ZUUL::> If-Modified-Since  Sun, 09 Jun 2013 01:10:31 GMT
REQUEST_DEBUG::ZUUL:: > GET  /?null HTTP/1.1
REQUEST_DEBUG::ZUUL::>
REQUEST_DEBUG::ORIGIN_RESPONSE:: > Date, Sun, 09 Jun 2013 02:09:19 GMT
REQUEST_DEBUG::ORIGIN_RESPONSE:: > Keep-Alive, timeout=5, max=100
REQUEST_DEBUG::ORIGIN_RESPONSE:: > ETag, "90a0-4deae5495ba47"
REQUEST_DEBUG::ORIGIN_RESPONSE:: > Expires, Sun, 09 Jun 2013 03:09:19 GMT
REQUEST_DEBUG::ORIGIN_RESPONSE:: > Cache-Control, max-age=3600
REQUEST_DEBUG::OUTBOUND: >  Date:Sun, 09 Jun 2013 02:09:19 GMT
REQUEST_DEBUG::OUTBOUND: >  Keep-Alive:timeout=5, max=100
REQUEST_DEBUG::OUTBOUND: >  ETag:"90a0-4deae5495ba47"
REQUEST_DEBUG::OUTBOUND: >  Expires:Sun, 09 Jun 2013 03:09:19 GMT
REQUEST_DEBUG::OUTBOUND: >  Cache-Control:max-age=3600
```

## 마치며
- zuul을 이용해서 proxy 기능을 사용할 때 request, response로 전달되고 있는 데이터가 궁금한 경우
해당 방식을 이용해서 사용 하면 된다.
- [developer guide](https://github.com/Netflix/zuul/wiki/zuul-simple-webapp) 에서 제공하고 있어서,
java 형식에 맞게 변경하였다.
- logging을 엄청 많이 하기에, develop 환경에만 적용해서 사용하기를 권장 한다. 
- zuul에 버전이 1.x인 경우 해당 방식을 이용하고 2.x인 경우는 조금 [다른 방식](https://github.com/Netflix/zuul/wiki/Filters) 을 이용해야 한다.

 