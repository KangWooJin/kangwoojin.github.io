---
title: "[JPA] EntityListeners 에서 DI를 하는 방법" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-07-12 09:00:00+09:00 
excerpt: Jpa에 있는 EntityListeners에 DI를 하는 방법에 대해서 알아보자.
---

## 들어가며
[Spring DI 순서]({% post_url spring/2020-07-12-spring-dependency-injection %}) 
에 대해서 기본적으로 알고 있어야 해당 내용을 이해하는데 도움이 된다!

따라서 이전 글을 읽고 이 글을 읽는게 조금이라도 도움이 될거라 생각한다.

JPA에서 Entity에 lifeCycle과 관련된 이벤트들이 존재하는데, 
해당 Event를 listen해주는 클래스가 EntityListeners 이다.

해당 EntityListeners를 이용하면, Spring에서 사용 가능한 DI도 사용할 수 있어서
POJO인 Entity에서 사용하는 것 보다 더 확장해서 사용할 수 있게 된다.

그렇지만, Spring과 JPA에서 bean을 생성하는 순서가 다르기 때문에 제대로 알고
사용해야 DI가 제대로 동작하게 된다.

## 테스트 환경

```java
@Data
@Entity
@EntityListeners(PersonEntityListener.class)
public class Person {
    @Id @GeneratedValue
    private Long id;

    private LocalDateTime createdTime;
}
```

- `Person` 이라는 `Entity`를 생성하고, `@EntityListeners`를 통해서 `EventListener` 클래스를 지정해 준다.

```java
public interface PersonRepository extends JpaRepository<Person, Long> {
}
```

```java
public class PersonEntityListener {
    @Autowired
    private PersonRepository personRepository;
    @PrePersist
    public void prePersist(Person person) {
        System.out.println("prePersist : " + personRepository);
        person.setCreatedTime(LocalDateTime.now());
    }
}
```

- `PersonEntityListener`에서는 `PersonRepository`를 `Autowired`로 주입 받을 수 있게 셋팅 한다.

```java
@SpringBootTest
class EventListenerTest {

    @Autowired
    PersonRepository personRepository;

    @Test
    void prePersistTest() {
        Person person = new Person();
        Person actual = personRepository.save(person);

        assertThat(actual.getCreatedTime()).isNotNull();
    }
}
```

- 전체 context가 load되는 것을 확인하기 위해 `@SpringBootTest`를 이용하고, Person을 save하여
`@PrePersist`가 잘 동작되는지 확인한다.

## Bean 생성 과정

- Spring Data Jpa는 application이 시작할 때 `EntityManagerFactory`를 먼저 bean으로 등록 한다.

### `EntityManagerFactory`가 무엇이길래?

- `EntityManagerFactory`는 `EntityManager`를 생성해주는 클래스이다.
- `EntityManagerFactory`는 `JpaRepository`를 Bean으로 등록할 때 반드시 필요한 클래스이다.
- `JpaRepository`는 `SimpleJpaRepository`를 기본 클래스로 사용하게 되는데, 해당 클래스에 `EntityManager`가 필요하게 되어 있다.

```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> implements JpaRepositoryImplementation<T, ID> {
	private final JpaEntityInformation<T, ?> entityInformation;
	private final EntityManager em;
	private final PersistenceProvider provider;
...
```

- `EntityManager`는 `SharedEntityManagerCreator`에 의해서 생성이 되는데, `EntityManagerFactory`가 필수 값이다.

```java
public static EntityManager createSharedEntityManager(EntityManagerFactory emf) {
    return createSharedEntityManager(emf, null, true);
}
```

### `EntityManagerFactory`랑 무슨 상관이지?

- `EntityManagerFactory`를 Bean으로 등록할 때 `EntityListener`에 대해서 Bean으로 등록하는 작업이 존재 한다.
- 그래서 `EntityListener`에서 `EntityManagerFactory`를 사용하는 Bean이 존재하게 되면 문제가 발생하는 것이다!
- `Repository` 입장에서는 `EntityManagerFactory`가 application 시작시에 생성 되었을거라 생각했는데, 
`EntityManagerFactory` 생성하는 과정에 Repository를 Bean으로 등록하고 있어서 문제가 발생하게 된다.
- 여기서 가장 큰 문제는 컴파일 에러는 발생하지 않고, 런타임 에러가 발생하게 된다.
- 그 이유는 Jpa는 `SpringContainedBean`에 Bean을 등록하게 되는데, Bean Create에 실패하게 되면,
`newInstance`를 이용해서 객체를 생성하고 넘어가서 실제로 `EntityListener`가 Bean이 아니라 newInstance로 생성되어 버린다.
 
```java
private SpringContainedBean<?> createBean(
			Class<?> beanType, LifecycleOptions lifecycleOptions, BeanInstanceProducer fallbackProducer) {
  try {
    if (lifecycleOptions.useJpaCompliantCreation()) {
      return new SpringContainedBean<>(
        this.beanFactory.createBean(beanType, AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR, false),
				this.beanFactory::destroyBean);
    }
    ...
  }
  catch (BeansException ex) {
    ...
	try {
	  return new SpringContainedBean<>(fallbackProducer.produceBeanInstance(beanType));
	}
	...
  }
}
```


## `EntityListener`에서 DI는 어떻게?

- `EntityListener`는 Bean으로 등록되어 있기에 Spring의 DI를 사용할 수 있는 조건을 충족해 있다.
- 어떻게 하면 해당 문제를 우회해서 안전하게 사용할 수 있는지 알아보자.

### 방법 1 - `ApplicationContext`

```java
public class PersonEntityListener {
    @Autowired
    private ApplicationContext applicationContext;
    @PrePersist
    public void prePersist(Person person) {
        PersonRepository personRepository = applicationContext.getBean(PersonRepository.class);
        System.out.println("prePersist : " + personRepository);
        person.setCreatedTime(LocalDateTime.now());
    }
}
```

- JpaRepository를 직접 주입받지 말고 `ApplicationContext`를 주입 받아서,
 `getBean`을 통해서 `Repository`를 가져오는 방법을 이용해도 된다.
- 해당 방식은 사용하는 시점에 Bean을 가져오기에, `EntityManagerFactory`가 생성이 되어 있으니 문제가 발생하지 않는다.

### 방법 2 - `@Lazy`

```java
public class PersonEntityListener {
    @Lazy
    @Autowired
    private PersonRepository personRepository;
    @PrePersist
    public void prePersist(Person person) {
        System.out.println("prePersist : " + personRepository);
        person.setCreatedTime(LocalDateTime.now());
    }
}
```

- `@Lazy`를 추가하여, context refresh 시점에는 proxy 상태였다가, 해당 Repository가 처음 사용될 때 초기화가 될 수 있게 변경 한다.
- 이렇게 하면 `EntityManagerFactory`가 생성된 이후기 때문에 문제가 발생하지 않는다.

### 방법 3 - `BootstrapMode` Deferred or Lazy
- BootstrapMode를 Deffrred로 설정하게 되면, JpaRepositories를 proxy로 생성 해준다.
- 또한, Spring context가 load하는 thread와 다른 thread를 이용해서 작업이 진행되고,
`ContextRefreshedEvent`에 trigger에 의해서 repository가 초기화가 진행된다.
- 결론은 `@Lazy`와 비슷하게 동작 하지만 application이 시작 전에 `Repository`들이 초기화가 보장되어 있고, load 속도도 빨라진다.
- BootstrapMode를 변경하는 방법은 `@EnableJpaRepositories`과, properties를 이용해서 설정 가능하다.
- Lazy의 경우는 앞에 설명한 방식을 전체 `Repository`에 일괄 적용 해주게 된다.
- Lazy로 Application을 시작하는 경우 런타임시에 문제가 발생할 수 있으니 주의해야 한다.

```java
@EnableJpaRepositories(bootstrapMode = BootstrapMode.DEFERRED)
```

```yaml
spring:
  data:
    jpa:
      repositories:
        bootstrap-mode: deferred
```

- 참고 : [BootstrapMode](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.bootstrap-mode)


## 마치며
- `EventListener`에 관해서 검색하면 static한 `ApplicationContext`를 이용해서 사용하라는 답변이 대부분인데,
동작 방식을 알고나면 불필요한 과정이 필요 없어지게 되는 거 같다.
- 원래 이 부분에 대해서 잘 몰랐지만 회사 동료분이 알기 쉽게 설명을 해주었고(설명 감사합니다.😍), 그걸 다시 한번 더 찾아서 정리 하였다.
- 추가적으로 2.3.0에서 부터는 Jpa bootstrapMode가 `Default`에서 `Deferred` 변경 되었다.
- 즉, 2.1.0 이상에서 JPA를 사용하고 있다면 `Deferred` 변경하는 것을 추천한다. [performance](https://spring.io/blog/2018/09/27/what-s-new-in-spring-data-lovelace)
- 중요한 부분은 JPA에서 제일 처음 생성되는 bean은 EntityManagerFactory이며, 해당 객체를 bean으로 등록할 때 EventListener들도 
같이 Bean으로 등록하게 된다.
- EventListener에서 JpaRepository를 주입하고 있다면, EntityManagerFactory가 생성이 되지 않았으니, 당연히 에러가 발생한다.

- - -  

[EventListenr](https://github.com/KangWooJin/spring-study/tree/master/querydsl/src/main/java/kangwoojin/github/io/querydsl/eventlistener)
관련 example code는 github에 올려두었으니 참고~!