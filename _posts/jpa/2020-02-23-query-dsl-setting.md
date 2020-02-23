---
title: "[JPA] Spring boot, gradle에서 query dsl 셋팅하기"
categories:
  - Programing
tags:
  - JPA
toc: true
toc_sticky: true
date: 2020-02-23 15:20:00+09:00 
last_modified_at: 2020-02-23 15:20:00+09:00
excerpt: Spring boot, gradle 환경에서 jpa의 객체지향 쿼리 언어인 query dsl을 셋팅 하기.
---

## 들어가며
spring boot, gradle에서 query dsl을 사용할 수 있게 간단하게 셋팅하는 방법과
컴파일시에 query dsl에서 생성 해주는 Q객체가 제대로 생성되는지 확인 해보자.

## 개 환경

- intellij 
- spring boot - 2.2.4
- gradle - 6.0.1
- spring data jpa 
- h2 db
- query dsl - 4.2.2
- lombok
- java 11

## build.gradle 셋팅

기존 프로젝트 환경에 query dsl 의존성을 추가 해준다.

```groovy
plugins {
    id 'org.springframework.boot' version '2.2.4.RELEASE'
    id 'io.spring.dependency-management' version '1.0.9.RELEASE'
    id 'java'
}

group = 'springboot.gradle.querydsl'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    compileOnly 'org.projectlombok:lombok'
    runtimeOnly 'com.h2database:h2'
    annotationProcessor 'org.projectlombok:lombok'
    testImplementation('org.springframework.boot:spring-boot-starter-test') {
        exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
    }

    // query dsl
    def queryDSL = '4.2.2'
    compile("com.querydsl:querydsl-jpa:${queryDSL}")
    compile("com.querydsl:querydsl-apt:${queryDSL}:jpa")
    annotationProcessor("com.querydsl:querydsl-apt:${queryDSL}:jpa")
    annotationProcessor("org.hibernate.javax.persistence:hibernate-jpa-2.1-api:1.0.2.Final")
    annotationProcessor("javax.annotation:javax.annotation-api:1.3.2")
}

test {
    useJUnitPlatform()
}
```

- compile("com.querydsl:querydsl-jpa:${queryDSL}")
- compile("com.querydsl:querydsl-apt:${queryDSL}:jpa")
- annotationProcessor("com.querydsl:querydsl-apt:${queryDSL}:jpa")
- annotationProcessor("org.hibernate.javax.persistence:hibernate-jpa-2.1-api:1.0.2.Final")
- annotationProcessor("javax.annotation:javax.annotation-api:1.3.2")

## 테스트
Entity를 생성하여 Query dsl을 이용해서 실제로 Q 객체가 생성되고 있는지 확인해보자. 

User Entity 객체를 생성
```java
@Data
@Entity
public class User {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
}
```

User Entity 객체를 조회할 User service 생성
JpaRepository를 이용해도 되는 간단한 쿼리이지만 Query dsl이 정상 동작하는지 확인하기 위해
간단한 쿼리를 작성

```java
@RequiredArgsConstructor
@Service
public class UserService {
    private final QUser qUser = QUser.user;
    private final EntityManager entityManager;

    public User findById(Long id) {
        final JPAQuery<User> query = new JPAQuery<>(entityManager);

        query.from(qUser)
             .where(qUser.id.eq(id));

        return query.fetchOne();
    }
}
```

Junit5에서 테스트를 진행하여 실제로 query dsl이 실행되고 있는지 확인해보자.
반드시 Entity가 추가되었으면 gradle에 build를 통해서 Q 객체를 생성하자.

```java
@Transactional
@AutoConfigureTestEntityManager
@SpringBootTest
class UserServiceTest {
    @Autowired
    private TestEntityManager testEntityManager;
    @Autowired
    private UserService userService;

    @Test
    void findByIdTest() {
        // Given
        User user = new User();
        String name = "woojin";
        user.setName(name);

        User savedUser = testEntityManager.persist(user);

        // When
        User actual = userService.findById(savedUser.getId());

        // Then
        assertThat(actual).isNotNull();
        assertThat(actual.getName()).isEqualTo(name);
    }
}

```

- @Transactional 
  - Transactional 어노테이션을 추가하지 않으니 해당 에러가 발생하였다.
  `No transactional EntityManager found` TestEntityManager에서 getEntityManager를 호출할 때
  transaction이 적용되어 있는지 체크하는 부분이 있으니 해당 어노테이션을 반드시 붙여줘야 한다.
  @DataJpaTest에서는 @Transactional이 붙어 있어서 몰랐지만 TestEntityManager만 AutoConfigure할 때 주의해야 한다.


```java
/**
  * Return the underlying {@link EntityManager} that's actually used to perform all
  * operations.
	* @return the entity manager
  */
public final EntityManager getEntityManager() {
  EntityManager manager = EntityManagerFactoryUtils.getTransactionalEntityManager(this.entityManagerFactory);
  Assert.state(manager != null, "No transactional EntityManager found");
  return manager;
}
```

- @AutoConfigureTestEntityManager
  - TestEntityManager를 사용할 수 있게 주입하는 어노테이션  
    

### Q 객체 생성 확인

![query_dsl_result](/assets/images/jpa/query_dsl_result.png)

## Error:java: Attempt to recreate a file for type
해당 에러가 발생하는 경우는 Q 객체를 생성해야 하는데 이미 폴더나 객체가 생성되어 있어 덮어 쓸 수 없을 때 발생한다.
따라서 build 되었을 때 생성되는 파일, out으로 생성되는 파일을 지우고 다시 실행하게 되면 정상적으로 동작한다.

- [https://qiita.com/kakasak/items/ae6a1335d68e06172d4d](https://qiita.com/kakasak/items/ae6a1335d68e06172d4d)

## Intellij의 AnnotationProcessors 활성화
![annotationProcessors](/assets/images/jpa/annotationProcessors.png)

해당 옵션을 활성화 해주지 않으면 lombok, Q 객체가 활성화 되지 않으니 반드시 실행 전에 활성화 해야 한다.

## 마치며
query dsl을 셋팅하는데 여러 가지 방법이 존재하는데 해당 방식으로 설정하게 되면
최초 한번 build 후 컴파일 시에도 어노테이션 프로세서를 통해서 Q 객체를 생성 해준다.

좀 더 원하는 디렉토리로 Q객체를 생성하고 싶으면 gradle에서 추가 옵션을 설정해서 
원하는 위치에 생성될 수 있도록 해줘야 한다.

### [query dsl demo - github](https://github.com/KangWooJin/jpa_query_dsl)



 