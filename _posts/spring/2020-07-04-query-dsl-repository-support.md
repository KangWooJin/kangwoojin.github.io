---
title: "[Querydsl] QuerydslRepositorySupport" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-07-04 08:00:00+09:00 
excerpt: JpaRepository를 확장해보고, QuerydslRepositorySupport를 이용해 custom query를 작성해 보자.
---

## 들어가며
JpaRepository를 확장해보고, QuerydslRepositorySupport를 이용해 custom query를 작성해 보자.

## JpaRepository 생성 하기

```java
@Entity
@Getter
@Setter
public class Campaign {
    @Id
    @GeneratedValue
    private Long id;

    private String name;

    private Long amount;

    @OneToMany(cascade = CascadeType.PERSIST, mappedBy = "campaign")
    @JsonBackReference
    private List<Event> events = new ArrayList<>();
}
```

- 테스트에 사용할 `Campaign`으로 Entity를 하나 생성 한다.

```java
public interface CampaignRepository extends JpaRepository<Campaign, Long> {
}
```

- `Campaign`에 대해서 `JpaRepository`를 생성 한다.
 
## JpaRepository 확장 하기

```java
public interface CampaignCustomRepository {

    Campaign findByName(String name);
}
```

- custom하게 사용할 `interface` 기반 `Repository`를 생성 한다.

```java
public interface CampaignRepository extends JpaRepository<Campaign, Long>, CampaignCustomRepository {
}
```

- 그 후 `CampaignRepository`에 상속 시켜 준다.

```java
public class CampaignCustomRepositoryImpl extends QuerydslRepositorySupport implements CampaignCustomRepository {
    private static final QCampaign qCampaign = QCampaign.campaign;

    public CampaignCustomRepositoryImpl() {
        super(Campaign.class);
    }

    @Override
    public Campaign findByName(String name) {
        return from(qCampaign)
                .select(qCampaign)
                .where(qCampaign.name.eq(name))
                .fetchOne();
    }
}
```

- `CampaignCustomRepository`를 구현하는 `CampaignCustomRepositoryImpl`를 생성 한다.
- 확장을 하는데 있어서 `QuerydslRepositorySupport`을 사용할 필요는 없지만 해당 Querydsl을 사용하고 있을 때 사용하기 좋다.
- `QuerydslRepositorySupport`을 상속하게 되면 어떤 type의 entity인지 super에 반드시 적어줘야 한다.
- `CampaignCustomRepository`을 상속하여 해당 메소드를 구현해 주면 된다.

## 테스트

```java
@RequiredArgsConstructor
@DataJpaTest
@TestConstructor(autowireMode = AutowireMode.ALL)
class CampaignCustomRepositoryTest {
    private final CampaignRepository campaignRepository;
    private final TestEntityManager testEntityManager;

    @Test
    void findByNameTest() {
        Campaign campaign = new Campaign();
        String name = "campaign";
        campaign.setName(name);
        Campaign savedCampaign = testEntityManager.persist(campaign);

        testEntityManager.flush();testEntityManager.clear();

        Campaign actual = campaignRepository.findByName(name);

        assertThat(actual.getId()).isEqualTo(savedCampaign.getId());
        assertThat(actual.getName()).isEqualTo(savedCampaign.getName());
    }
}
``` 

```sql
Hibernate: 
    /* select
        campaign 
    from
        Campaign campaign 
    where
        campaign.name = ?1 */ select
            campaign0_.id as id1_0_,
            campaign0_.amount as amount2_0_,
            campaign0_.name as name3_0_ 
        from
            campaign campaign0_ 
        where
            campaign0_.name=?
```

- spring boot 2.2.0 부터 Test에서 생성자 주입이 가능하게 변경되어 `TestConstructor`을 사용하면 생성자 주입이 가능하다.
- `DataJpaTest`을 통해서 `Repository` layer 만 주입 받도록 한다.

## 주의사항

- `JpaRepository`를 확장할 때 구현 class의 이름은 `Impl`로 끝나야 한다.
- 만약 `Impl`로 끝나지 않는 경우 `@EnableJpaRepositories(repositoryImplementationPostfix = "Impl")`을 통해서 postFix를 변경해줘야 한다.

## 더 나아가기
- `QuerydslRepositorySupport`는 `JPQLQuery`를 사용하기에 `from`으로 시작 되어야 하는 제약조건이 있다.
- 보통 쿼리는 `select`로 시작하기도 하고 Querydsl을 사용할 때 `JPAQuery`을 사용 하기에, `JPAQuery`로 사용할 수 있게 변경 해보자.

```java
public abstract class QuerydslCustomRepositorySupport extends QuerydslRepositorySupport {
    private JPAQueryFactory queryFactory;

    public QuerydslCustomRepositorySupport(Class<?> domainClass) {
        super(domainClass);
    }

    @Autowired
    @Override
    public void setEntityManager(EntityManager entityManager) {
        super.setEntityManager(entityManager);
        this.queryFactory = new JPAQueryFactory(entityManager);
    }

    protected <T> JPAQuery<T> select(Expression<T> expr) {
        return queryFactory.select(expr);
    }

    protected <T> JPAQuery<T> selectFrom(EntityPath<T> from) {
        return queryFactory.selectFrom(from);
    }

}
```

- `QuerydslCustomRepositorySupport`을 만들어서 `QuerydslRepositorySupport` 대신 상속받아서 사용하는 방법도 있다.
- 주의해야 할 부분은 `setEntityManager`에 `@Autowired`를 통해서 `EntityManager`을 주입 받아야 한다.
- `select`로 시작하고 싶은 사람만 사용하되, 실제로 `QuerydslRepositorySupport`도 `JPAQuery`를 생성 하긴 한다.
 
## 마치며
- `Service` layer와 `Repository` layer에서 querydsl을 사용하고 있는데, 컨벤션도 안맞는거 같고
`Service` layer에 경우 `DataJpaTest`를 사용하여 테스트가 불가능하다.
- query 관련된 부분은 `Repository`에서 처리하고 business 로직들은 `Service` layer에서 처리하는게 좋다고 생각한다.


- - - 
[flushModeTest](https://github.com/KangWooJin/spring-study/blob/master/querydsl/src/test/java/kangwoojin/github/io/querydsl/FlushModeTest.java)
관련 example code는 github에 올려두었으니 참고~!
 