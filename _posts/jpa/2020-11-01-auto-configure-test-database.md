---
title: "[JPA] DataJpaTest: 테스트시에 실제 DB 사용하기" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-10-31 12:00:00+09:00 
excerpt: AutoConfigureTestDatabase에 대해서 알아보자.
---

## 들어가며
- Jpa관련 테스트를 할때 `@DataJpaTest`를 이용해서 진행하면 필요한 Configuration만 주입받아서 빠르게 테스트를 진행할 수 있고 롤백도 되어서
간단하게 결과를 확인할 수 있는 장점이 있다.
- `DataJpaTest`를 설정했을 때 장점도 있지만, 실제 DB를 이용해서 테스트를 해보려고 하니, 에러가 발생하면서 동작하지 않아 삽질한 과정을 기록하려고 한다.
- 테스트에서 메모리 DB인 H2 데이터베이스를 이용해서 테스트하는게 아니라 Mysql 처럼 물리 데이터 베이스를 이용해서 테스트를 할때 어떻게 해야 하는지 알아보자.

## DataJpaTest
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@BootstrapWith(DataJpaTestContextBootstrapper.class)
@ExtendWith(SpringExtension.class)
@OverrideAutoConfiguration(enabled = false)
@TypeExcludeFilters(DataJpaTypeExcludeFilter.class)
@Transactional
@AutoConfigureCache
@AutoConfigureDataJpa
@AutoConfigureTestDatabase
@AutoConfigureTestEntityManager
@ImportAutoConfiguration
public @interface DataJpaTest {
}
```
- `DataJpaTest`를 사용하면 자동으로 `AutoConfigure` 해주는 옵션이 상당히 많이 붙어 있다.
- 이 중에서도 `AutoConfigureTestDatabase` 부분이 무엇을 담당 해주는지 알아보자.

## AutoConfigureTestDatabase

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@ImportAutoConfiguration
@PropertyMapping("spring.test.database")
public @interface AutoConfigureTestDatabase {
  @PropertyMapping(skip = SkipPropertyMapping.ON_DEFAULT_VALUE)
  Replace replace() default Replace.ANY;

  EmbeddedDatabaseConnection connection() default EmbeddedDatabaseConnection.NONE;

  enum Replace {
    ANY,
	AUTO_CONFIGURED,
	NONE
  }
}
```

- 해당 annotation을 보게 되면 `replace`, `connection`에 대해서 변경이 가능하다.
- `replace` 설정에 따라서 embeddedDatabase를 할지 안할지를 선택하게 된다.
- 여기서 눈여겨봐야할 점은 `@PropertyMapping("spring.test.database")` 이 부분이다.

## PropertyMapping
- `PropertyMapping`에 대해서 모르면 다음으로 넘어갈 때 이해하기 어렵기 때문에 해당 부분에 대해서도 알아본다.

```java
@Retention(RUNTIME)
@PropertyMapping("my.example")
public @interface Example { 
    String name();
}

@Example(name="Spring")
public class MyTest {
}
```
- `PropertyMapping`에 설명을 가져온 부분인데, 해당 annotation처럼 사용하게 된다면 `my.example.name` 이라는 property의 값을 사용하게 된다.
- 즉, MyTest에서 `my.example.name`의 값은 `Spring`이 되게 된다.

## TestDatabaseAutoConfiguration 

```java
@Configuration(proxyBeanMethods = false)
@AutoConfigureBefore(DataSourceAutoConfiguration.class)
public class TestDatabaseAutoConfiguration {
  @Bean
  @ConditionalOnProperty(prefix = "spring.test.database", name = "replace", havingValue = "AUTO_CONFIGURED")
  @ConditionalOnMissingBean
  public DataSource dataSource(Environment environment) {
    return new EmbeddedDataSourceFactory(environment).getEmbeddedDatabase();
  }

  @Bean
  @ConditionalOnProperty(prefix = "spring.test.database", name = "replace", havingValue = "ANY",
		matchIfMissing = true)
  public static EmbeddedDataSourceBeanFactoryPostProcessor embeddedDataSourceBeanFactoryPostProcessor() {
    return new EmbeddedDataSourceBeanFactoryPostProcessor();
  }
}
```
- `TestDatabaseAutoConfiguration`에 보면 `ConditionalOnProperty`으로 property의 값이 어떤 값이면 해당 bean이 등록될 수 있게 되어 있다.
- 앞에서 봤던 `AutoConfigureTestDatabase`에 replace 값에 따라서 어떤 bean이 등록될지 설정되는 것이다.
- 기본값은 `spring.test.database.replace: any`로 셋팅 되므로 아래쪽 `EmbeddedDataSourceBeanFactoryPostProcessor`가 Bean으로 등록이 된다.


## EmbeddedDatabase

```java
private static class EmbeddedDataSourceFactory {
  private final Environment environment;
  EmbeddedDataSourceFactory(Environment environment) {
    this.environment = environment;
  }

  EmbeddedDatabase getEmbeddedDatabase() {
    EmbeddedDatabaseConnection connection = this.environment.getProperty("spring.test.database.connection",
				EmbeddedDatabaseConnection.class, EmbeddedDatabaseConnection.NONE);
    if (EmbeddedDatabaseConnection.NONE.equals(connection)) {
      connection = EmbeddedDatabaseConnection.get(getClass().getClassLoader());
    }
    Assert.state(connection != EmbeddedDatabaseConnection.NONE,
				"Failed to replace DataSource with an embedded database for tests. If "
						+ "you want an embedded database please put a supported one "
						+ "on the classpath or tune the replace attribute of @AutoConfigureTestDatabase.");
    return new EmbeddedDatabaseBuilder().generateUniqueName(true).setType(connection.getType()).build();
  }
}
```

```java
public enum EmbeddedDatabaseConnection {
  NONE(null, null, null),
  H2(EmbeddedDatabaseType.H2, "org.h2.Driver", "jdbc:h2:mem:%s;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE"),
  DERBY(EmbeddedDatabaseType.DERBY, "org.apache.derby.jdbc.EmbeddedDriver", "jdbc:derby:memory:%s;create=true"),
  HSQL(EmbeddedDatabaseType.HSQL, "org.hsqldb.jdbcDriver", "jdbc:hsqldb:mem:%s");
}
```
- `EmbeddedConnection`으로 제공하고 있는것은 `H2`, `DERBY`, `HSQL`을 제공하고 있고 `NONE`의 경우는 에러를 발생하게 된다.
- `EmbeddedDatabaseConnection.get(getClass().getClassLoader());` 부분에서 추가된 라이브러리에 따라서 설정되는 값이 정해진다.
- 만약 H2를 추가했으면 H2가 되고, DERBY면 DERBY가 default가 된다. 흠.. 2개를 다하면? 어떻게 될지는 안해봤다.

## 실제 database(ex: mysql)는 어떻게 설정하나?
- `@DataJpaTest`를 사용하면 자동으로 `EmbeddedDatabase`를 사용해버리기 때문에, mysql을 설정을 해버릴 수가 없다.
- 방법은 바로 `AutoConfigureTestDatabase`를 덮어써서 해당 설정이 동작하지 않게 바꿔버리면 된다.

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
class AutoConfigureTestDatabaseTest {
}
```

- `(replace = Replace.NONE)`를 통해서 `TestDatabaseAutoConfiguration`에서 `DataSource`가 bean으로 등록되지 않게 하면
`DataSourceAutoConfiguration`에 의해서 `DataSource`가 등록되게 된다.
- 그러면 property에 설정한 dataSource의 설정 값을 확인하여 적절한 DataSource를 생성하게 된다. SpringBoot 2.0 이상부터는 `HikariDataSource`가 Default로 등록이 되게 된다.
- 이런식으로 `DataJpaTest`를 이용하면서 실제 데이터베이스로 테스트를 해보기 위해서는 `AutoConfigureTestDatabase`를 이용해서 EmbeddedDatabase
설정이 되지 않게 해주어야 한다.
     
## 마치며
- `DataJpaTest`를 사용 하면서도 dataSource만 바꾸면 쉽게 원하는 db에 붙을 수 있을거라 생각했는데, 생각보다 복잡한 설정에 의해서
자동으로 셋팅이 되고 있었던 것이였다..
- 역시 문제가 생겨야 하나씩 더 찾아보고 배워나가는거 같다..ㅎ
- `SpringBootTest`를 사용하게 되면 이러한 현상이 발생하지 않는다. 왜 발생안하는지는 Annotation을 잘 살펴보면 된다!
- 참고 : [https://docs.spring.io/spring-boot/docs/1.1.0.M1/reference/html/howto-database-initialization.html](https://docs.spring.io/spring-boot/docs/1.1.0.M1/reference/html/howto-database-initialization.html)