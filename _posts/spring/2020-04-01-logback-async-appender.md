---
title: "[Spring] Logback에 AsyncAppender에 대해 알아보자"
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-04-02 00:00:00+09:00 
excerpt: Spring Boot에서 logback을 이용해 AsyncAppender에 대해서 알아보자.
---

## 들어가며
우리 팀에서는 `Spring boot`에 `Logback`의 `RollingFileAppender`를 이용해서 로그를 남기고 있다.

이전 팀에서는 `RollingFileAppender`에 `AsyncAppender`로 한번 더 감싸서 전송을 하고 있는 것을 확인하고

AsyncAppender를 적용했을 때 문제가 있을지 장점은 무엇인지에 대해서 알아보려고 한다.

## AsyncAppender 셋팅하기
`Logback`에서 지원하는 Appender 중의 하나이며 기존에 사용하고 있는 Appender를 레퍼런스하여 
사용할 수 있도록 제공하고 있다.

```xml
<configuration debug="true">
  <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />

  <property name="LOG_PATH" value="/tmp"/>
  <property name="LOG_FILE" value="${LOG_PATH}/logback-test.log"/>

  <appender name="RollingFile" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_FILE}</file>
    <encoder>
      <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS,Asia/Tokyo} %-5level [%thread,%X{X-B3-TraceId:-},%X{X-B3-SpanId:-},%X{X-Span-Export:-}] %logger{36} [%file:%line] - %msg ##%n</Pattern>
    </encoder>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}</fileNamePattern>
      <maxHistory>5</maxHistory>
    </rollingPolicy>
  </appender>

  <root level="TRACE">
    <appender-ref ref="RollingFile" />
  </root>

</configuration>
```

spring boot부터는 resources에 logback에 configuration을 포함하는 파일이 존재하면 해당 파일을 
load 하여 셋팅해 준다.

- logback-spring.xml
- logback.xml
- logback.groovy
- logback-spring.groovy

`logback-spring.xml` 파일을 생성하여 `RollingFileAppender`를 셋팅 해준다.

```xml
<configuration debug="true">
  <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />

  <property name="LOG_PATH" value="/tmp"/>
  <property name="LOG_FILE" value="${LOG_PATH}/logback-test.log"/>

  <appender name="RollingFile" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_FILE}</file>
    <encoder>
      <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS,Asia/Tokyo} %-5level [%thread,%X{X-B3-TraceId:-},%X{X-B3-SpanId:-},%X{X-Span-Export:-}] %logger{36} [%file:%line] - %msg ##%n</Pattern>
    </encoder>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}</fileNamePattern>
      <maxHistory>5</maxHistory>
    </rollingPolicy>
  </appender>

  <!-- AsyncAppender 추가 -->
  <appender name="file-async" class="ch.qos.logback.classic.AsyncAppender">
    <appender-ref ref="RollingFile" />
    <queueSize>1</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <includeCallerData>false</includeCallerData>
    <neverBlock>false</neverBlock>
  </appender>

  <root level="TRACE">
    <!-- AsyncAppender로 변경 -->
    <appender-ref ref="file-async" />
  </root>

</configuration>

```

그 후 `AsyncAppender`를 추가하여 `appender-ref`에 `RollingFileAppender`를 연결해 준다.

그리고 root에서 `AsyncAppender`로 셋팅 해주면 `RollingFileAppender`를 async하게 사용할 수 있다.

## AsyncAppender 설정 확인
`AsyncAppender`에서는 제공하는 옵션이 딱 5가지 밖에 존재 하지 않는다.

### queueSize
async로 동작하기 위해서 log들을 BlockingQueue를 이용해 버퍼에 저장해 둔다.

버퍼에 저장해두는 queue의 사이즈를 의미하며 해당 queue의 사이즈의 80%가 초과하게 되면

log level의 warn, error를 제외하고는 drop하는 설정을 갖고 있다.

따라서 queueSize를 잘 확인하고 셋팅 해야 한다.

### discardingThreshold
해당 부분이 queue가 어느정도 채워지면 몇퍼센트를 드랍할지를 결정하는 부분인데 기본값이 20으로 되어 있다.

0으로 셋팅하게 되면 drop되지 않게 셋팅이 가능하다.

### includeCallerData
```java
2020-04-01 23:52:24.002 WARN  [http-nio-8080-exec-1,,,] k.github.io.demo.service.LogbackTest [LogbackTest.java:26] - http-nio-8080-exec-1 ##
2020-04-01 23:52:24.002 ERROR [http-nio-8080-exec-1,,,] k.github.io.demo.service.LogbackTest [LogbackTest.java:27] - http-nio-8080-exec-1 ##
```

해당 설정을 이용하면 로그를 남길때 현재 클래스에 어느 라인 수에서 로그가 남기고 있는지 알 수 있게 된다.
`LogbackTest.java:26` 라고 나오게 되며 해당 옵션을 끄게 되면 드라마틱한 성능 향상을 가져올 수 있다.
대신에 로그가 어디서 찍히는지 추적하기가 어려워지는 문제가 있게 된다.
 
### maxFlushTime
>Depending on the queue depth and latency to the referenced appender, the AsyncAppender may take an unacceptable amount of time to fully flush the queue. When the LoggerContext is stopped, the AsyncAppender stop method waits up to this timeout for the worker thread to complete. Use maxFlushTime to specify a maximum queue flush timeout in milliseconds. Events that cannot be processed within this window are discarded. Semantics of this value are identical to that of Thread.join(long).

설명을 읽고 값을 변경해가면서 적용해보았지만 의미를 잘 모르겠어서 해당 설명을 추가 해둔다.
 
### neverBlock
queue에 가득차게 되는 경우 다른 쓰레드의 작업들이 blocking 상태에 빠지게 되는데
해당 옵션을 true하게 되면 blocking 상태에 빠지지 않고 log를 drop하며 계속 진행할 수 있게 해준다.

## 왜 사용할까
기존에 sync 방식의 `RollingFileAppender`를 이용해도 서비스를 동작하는데는 아무 문제가 없다.

그렇지만 `AsyncAppender`를 이용하게 되면 장점이 존재하기에 해당 방식으로 변경해보려고 한다.

내가 생각하기엔 장점은 딱 한가지인 것 같다.
- 기존 sync 방식보다 로그를 남기는데 있어서 성능상 빨라진다.

하지만 단점이랑 고려해야 하는 사항들이 생각보다 많이 있다.
- queueSize를 너무 작게 하는 경우 WARN, ERROR를 제외하고 로그의 손실을 가져올 수 있다.
- 버퍼를 이용하니 메모리의 사용량이 증가하고 CPU 사용량 또한 증가한다.
- 중간에 서버가 shutdown되는 경우에 버퍼에 로그가 남아 있으면 버퍼의 로그를 다 쓰기 전에 종료되어 손실이 발생한다.

대용량의 트래픽을 받으면서 로그를 많이 남겨야 하는 서비스의 경우는 `AsyncAppender`를 이용해서
성능상의 이점을 가져오는게 더 좋을 거 같다.
 
## 마치며
기존에 잘 동작하고 있던 서비스에 `AsyncAppender`가 성능상의 이점을 가져오니 적용해야겠다 라고만 생각한다면
한번 더 고려를 해보는 것이 좋을 것 같다. 

로그의 손실이 발생할 수 있고 버퍼를 사용하니 메모리 사용과 cpu 사용이 증가할 수 있는 문제가 있다.

정말로 서비스의 로그가 많이 남기고 있으며, 로그의 손실이 발생하더라도 크리티컬하지 않는다면 적용해도 괜찮아 보인다.

로그의 버퍼가 꽉차서 application이 blocking되지 않기 위해 반드시 `<neverBlock>true</neverBlock>`
를 적용하기를 바란다!


- - - 

 