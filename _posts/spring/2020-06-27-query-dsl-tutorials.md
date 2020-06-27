---
title: "[Querydsl] Querydsl tutorials - querying" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-06-27 08:00:00+09:00 
excerpt: Querydsl을 이용해 간단한 질의 테스트를 해보자.
---

## 들어가며
Querydsl의 프로젝트 셋팅 관련해서는 무려 3번이나 포스팅 했는데, 실제로 사용하는 포스팅에 대해서는
없어 사용법에 대해서도 기록을 해두려고 한다.

아직 옮긴 조직에서 업무를 많이 할당 받아서 하는게 아니다 보니, 실제 Querydsl을 이용해 작업하는 부분이 없었다.

그렇지만 최근에 querydsl 관련 작업을 하려고 하니, 어떻게 작성해야 하는지 하나도 모른다는 걸 깨달았기에 
지금부터라도 천천히 example을 만들어둬서 연습을 해두려고 한다.

## Test Setting
- Querydsl을 이용해서 insert 부분은 지원하지 않고, select, update, delete 만 지원 한다.
- `Campaign`, `Event` class를 생성하였고, `1:N`에 양방향 맵핑관계로 테스트를 진행하였다.

```java
@Entity
@Data
public class Campaign {
    @Id
    @GeneratedValue
    private Long id;

    private String name;

    private Long amount;

    @OneToMany(cascade = CascadeType.PERSIST, mappedBy = "campaign")
    private List<Event> events = new ArrayList<>();
}
```

```java
@Entity
@Data
public class Event {
    @Id
    @GeneratedValue
    private Long id;

    private String name;
    private Long amount;

    @ManyToOne
    @JoinColumn(name = "campaign_id")
    private Campaign campaign;
}
```

- 테스트 환경은 `DataJpaTest`를 사용하여 `TestEntityManager`를 통해서 entity를 insert 한다.
 
```java
@DataJpaTest
class CampaignQueryDslTest {
    QCampaign qCampaign = QCampaign.campaign;
    QEvent qEvent = QEvent.event;
    @Autowired
    private TestEntityManager testEntityManager;
    EntityManager entityManager;

    @BeforeEach
    void setUp() {
        entityManager = testEntityManager.getEntityManager();
    }
}
```

## Select

- `Campaign`은 name, amount를 받아서 `generateCampaign`에서 생성 하도록 작성해 두었다.

```java
private Campaign generateCampaign(String name, Long amount) {
  Campaign campaign = new Campaign();
  campaign.setName(name);
  campaign.setAmount(amount);
  Event event = new Event();
  event.setAmount(5L);
  campaign.setEvents(List.of(event));
  return testEntityManager.persist(campaign);
}
```

```java
@Test
void selectByIdTest() {
    String name = RandomStringUtils.randomAlphabetic(5);
    long amount = RandomUtils.nextLong();
    Campaign campaign = generateCampaign(name, amount);
    JPAQuery<Campaign> query = new JPAQuery(entityManager);

    Campaign actual = query.select(qCampaign)
                           .from(qCampaign)
                           .where(qCampaign.id.eq(campaign.getId()))
                           .fetchOne();

    assertThat(actual).isNotNull();
    assertThat(actual.getName()).isEqualTo(name);
    assertThat(actual.getAmount()).isEqualTo(amount);
}
```

```sql
select campaign
from Campaign campaign
where campaign.id = ?1
```

## Update

```java
@Test
void updateTest() {
    String name = RandomStringUtils.randomAlphabetic(5);
    long amount = RandomUtils.nextLong();
    Campaign campaign = generateCampaign(name, amount);

    JPAQueryFactory jpaQueryFactory = new JPAQueryFactory(entityManager);
    long count = jpaQueryFactory.update(qCampaign)
                                .where(qCampaign.id.eq(campaign.getId()))
                                .set(qCampaign.name, "changedName")
                                .execute(); // db에 직접 실행

    assertThat(count).isEqualTo(1);

    testEntityManager.clear(); // entity manager에 cache되어 있어 clear해야 신규 반영된 데이터를 읽어 온다.

    JPAQuery<Campaign> query = new JPAQuery(entityManager);
    Campaign actual = query.select(qCampaign)
                           .from(qCampaign)
                           .where(qCampaign.id.eq(campaign.getId()))
                           .fetchOne();

    assertThat(actual).isNotNull();
    assertThat(actual.getName()).isEqualTo("changedName");
    assertThat(actual.getAmount()).isEqualTo(amount);
}
```

```sql
update Campaign campaign
set campaign.name = ?1
where campaign.id = ?2
```

## Delete
```java
@Test
void deleteTest() {
    String name = RandomStringUtils.randomAlphabetic(5);
    long amount = RandomUtils.nextLong();
    Campaign campaign = generateCampaign(name, amount);

    JPAQueryFactory jpaQueryFactory = new JPAQueryFactory(entityManager);
    long count = jpaQueryFactory.delete(qCampaign)
                                .where(qCampaign.id.eq(campaign.getId()))
                                .execute();

    assertThat(count).isEqualTo(1);

    testEntityManager.clear();

    JPAQuery<Campaign> query = new JPAQuery(entityManager);
    Campaign actual = query.select(qCampaign)
                           .from(qCampaign)
                           .where(qCampaign.id.eq(campaign.getId()))
                           .fetchOne();

    assertThat(actual).isNull();
}
```

```sql
delete from Campaign campaign
where campaign.id = ?1
```

## SubQuery

```java
@Test
void subQueryTest() {
    String name = RandomStringUtils.randomAlphabetic(5);
    long amount = 10L;
    Campaign campaign = generateCampaign(name, amount);

    JPAQuery<Campaign> query = new JPAQuery(entityManager);
    Campaign actual = query.select(qCampaign)
                           .from(qCampaign)
                           .where(qCampaign.amount.gt(
                             JPAExpressions.select(qEvent.amount)
                                           .from(qEvent)
                           ))
                           .fetchOne();

    assertThat(actual).isNotNull();
}
```

```sql
select campaign
from Campaign campaign
where campaign.amount > (select event.amount
                        from Event event)
)
```

## Ordering

```java
@Test
void orderingTest() {
    String name = RandomStringUtils.randomAlphabetic(5);
    long amount = RandomUtils.nextInt();
    Campaign c1 = generateCampaign(name, amount);
    Campaign c2 = generateCampaign(name, amount);

    JPAQuery<Campaign> query = new JPAQuery(entityManager);
    List<Campaign> actual = query.select(qCampaign)
                                 .from(qCampaign)
                                 .orderBy(qCampaign.id.desc())
                                 .fetch();

    assertThat(actual).hasSize(2);
    assertThat(actual.get(0).getId()).isEqualTo(c2.getId());
    assertThat(actual.get(1).getId()).isEqualTo(c1.getId());
}
```

```sql
select campaign
from Campaign campaign
order by campaign.id desc
```

## Grouping

```java
@Test
void groupingTest() {
    String name = RandomStringUtils.randomAlphabetic(5);
    long amount = RandomUtils.nextInt();
    generateCampaign(name, amount);
    generateCampaign(name, amount);

    JPAQuery<Campaign> query = new JPAQuery(entityManager);
    Long sum = query.select(qCampaign.amount.sum())
                    .from(qCampaign)
                    .groupBy(qCampaign.name)
                    .fetchOne();

    assertThat(sum).isEqualTo(amount * 2);
}
```

```sql
select sum(campaign.amount)
from Campaign campaign
group by campaign.name
```

## 마치며
아주 간단한 entity에서 쉬운 query에 대해서만 한번 테스트를 진행해 보았다.

querydsl을 이용해서 update, delete가 가능하다는 부분을 모르고 있었고, insert도 안된다는 거를 몰랐었다.
이번 기회에 querydsl을 사용한다면 기본적인 지식인 부분에 대해서 다시 한번 알아갈 수 있어서 좋았던 것 같다.

같은 `EntityManager`를 이용 하기에 entity 값이 캐시가 되어 있는 부분도 다시 한번 점검하는 기회가 된거 같다. 

- - - 
[querydslTest](https://github.com/KangWooJin/spring-study/blob/master/querydsl/src/test/java/kangwoojin/github/io/querydsl/CampaignQueryDslTest.java)
관련 example code는 github에 올려두었으니 참고~!
 