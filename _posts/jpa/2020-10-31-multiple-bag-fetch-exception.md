---
title: "[JPA] MultipleBagFetchException: cannot simultaneously fetch multiple bags" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-10-31 12:00:00+09:00 
excerpt: MultipleBagFetchException이 왜 발생하는지와 어떻게 해결하는지에 대해서 알아보자.
---

## 들어가며
JPA를 이용해서 하나의 Entity에 1:N 관계를 여러개 설정을 해서 사용하다 보면 발생하는 `MultipleBagFetchException`에 대해서 알아보자.

## 언제 발생하는가?

```java
@Data
@Entity
public class User {
  @Id
  @GeneratedValue
  private Long id;
  private String name;
  private String country;

  @ElementCollection(fetch = FetchType.LAZY)
  private List<String> nameHistories;

  @ElementCollection(fetch = FetchType.LAZY)
  private List<String> countryHistories;
}
```

- `User` Entity에 nameHistories, countryHistories처럼 1:N 관계가 형성되는 2개의 ElementCollection을 생성하였다.

```java
@DataJpaTest
class UserFetchJoinTest {
  static final QUser qUser = QUser.user;
  @Autowired
  TestEntityManager em;
  private User createUser() {
    User user = new User();
    user.setName("woojin");
    user.setCountry("KR");
    user.setCountryHistories(List.of("KR"));
    user.setNameHistories(List.of("woojin"));
    return em.persist(user);
  }

  @Test
   void elementCollectionSizeIsOne() {
   User user = createUser();
   JPAQuery<User> query = new JPAQuery<>(em.getEntityManager());

   User actual = query.select(qUser)
                      .from(qUser)
                      .leftJoin(qUser.nameHistories).fetchJoin()
                      .leftJoin(qUser.countryHistories).fetchJoin()
                      .where(qUser.id.eq(user.getId()))
                      .fetchOne();
   assertThat(actual.getCountryHistories()).hasSize(1);
   assertThat(actual.getNameHistories()).hasSize(1);
  }
}
```

- `User`를 생성하고, 조회할때 `nameHistories`, `countryHistories`를 fetchJoin을 통해서 `EAGER`로 가져오려고 할때 `MultipleBagFetchException`이 발생하게 된다.

## 왜 발생하는가?
- 조회할때 `BagType`의 Collection을 2개 이상 조회하려고 했기에 발생하였다.
- `BagType`이 무슨말인가? Collection에서 중복을 허용하는 Collection을 Bag이라는 것을 한번쯤은 들어봤을 것이다.

```java
public abstract class BasicLoader extends Loader {
  protected void postInstantiate() {
    Loadable[] persisters = getEntityPersisters();
    String[] suffixes = getSuffixes();
    descriptors = new EntityAliases[persisters.length];
    for ( int i=0; i<descriptors.length; i++ ) {
      descriptors[i] = new DefaultEntityAliases( persisters[i], suffixes[i] );
    }

    CollectionPersister[] collectionPersisters = getCollectionPersisters();
    List bagRoles = null;
    if ( collectionPersisters != null ) {
      String[] collectionSuffixes = getCollectionSuffixes();
      collectionDescriptors = new CollectionAliases[collectionPersisters.length];
      for ( int i = 0; i < collectionPersisters.length; i++ ) {
        if ( isBag( collectionPersisters[i] ) ) {
          if ( bagRoles == null ) {
            bagRoles = new ArrayList();
          }
          bagRoles.add( collectionPersisters[i].getRole() );
        }
        collectionDescriptors[i] = new GeneratedCollectionAliases(collectionPersisters[i],collectionSuffixes[i]);
      }
    }
    else {
      collectionDescriptors = null;
    }
    if ( bagRoles != null && bagRoles.size() > 1 ) {
      throw new MultipleBagFetchException( bagRoles );
    }
  }
}
```
- QueryPlan을 생성할 때 동작하는 부분인데, 마지막에 `bagRoles.size() > 1` 인 경우 `MultipleBagFetchException`를 발생 한다.
- 왜 이러한 에러를 발생시켰는지에 대해서는 잘 모르겠지만 추측되는 원인에 대해서 적어보겠다.
- 1:N, 1:M을 fetchJoin으로 가져오게 되면 query에 left outer join으로 query를 가져오게 되고, 결과가 N*M개의 데이터가 생성이 되어 중복 데이터가 발생하게 된다.
- List의 경우 중복 데이터를 허용하니, 필요로 했던 데이터 보다 많은 데이터가 생성이 될 것이기에 이러한 문제를 사전에 차단했지 않을까 싶다.

## 어떻게 해결하는가?
```java
@Data
@Entity
public class User {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    private String country;

    @ElementCollection
    private Set<String> nameHistories;

    @ElementCollection
    private Set<String> countryHistories;
}
```

- BagType이 발생하지 않게 `MapType`이나 `SetType`으로 변경하면 해당 에러가 발생하지 않고 정상적으로 query가 발생하게 된다.

## 마치며
- 원래 테스트 하려고 했던 것은 에러를 발생하지 않고 중복데이터를 발생시켰지만 조건이 다르다 보니 다른 에러가 발생하였다.
- 그렇지만 이러한 것도 하나의 또 다른 지식을 얻은거 같다 :D
- 참고 : [https://perfectacle.github.io/2019/05/01/hibernate-multiple-bag-fetch-exception/](https://perfectacle.github.io/2019/05/01/hibernate-multiple-bag-fetch-exception/)