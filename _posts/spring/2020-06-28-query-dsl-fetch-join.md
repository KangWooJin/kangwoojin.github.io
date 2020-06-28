---
title: "[Querydsl] Querydsl fetch join" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-06-28 09:00:00+09:00 
excerpt: Querydsl fetch join에 대해서 알아보자.
---

## 들어가며
JPA에서 fetchJoin을 사용할 때는 대부분 N+1을 해결할 때 많이 사용한다.

그렇지만 사람들마다 사용하는 방법이 다양했고, 사용하는 방법을 모르는 사람들은 복사 붙여넣거나
실제 query와 비슷하게 작성을 한다.

fetchJoin을 썼으니깐 N+1이 안생길꺼라고 생각할 수 있는데, 잘 써야지만 안생기기에 제대로 알고 써야 한다.

fetchJoin을 사용을 하면 어떤 변화가 생기는지 알아보자.

## 테스트 환경
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
    private List<Event> events = new ArrayList<>();
}
```

```java
@Entity
@Getter
@Setter
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

- `Campaign`과 `Event`는 1:N이며, 양방향 연관관계를 설정하였다.
- lombok의 `@Data`를 사용하고 양방향 관계를 사용하면 순환참조가 발생하니 `@Getter`, `@Setter`를 사용하던지
`@ToString`을 오버라이딩 해야 한다.
- 순환 참조를 해결하는 방법은 많으니 다른 방법도 괜찮다.

```java
private Campaign generateCampaign(String name, Long amount) {
    Campaign campaign = new Campaign();
    campaign.setName(name);
    campaign.setAmount(amount);
    Event event = new Event();
    event.setAmount(5L);
    campaign.setEvents(List.of(event));
    Campaign savedCampaign = testEntityManager.persist(campaign);
    event.setCampaign(campaign);
    return savedCampaign;
}
```

## Join only

```java
@Test
void notUsedFetchJoinTest() {
    String name = RandomStringUtils.randomAlphabetic(5);
    long amount = RandomUtils.nextLong();
    Campaign campaign = generateCampaign(name, amount);

    testEntityManager.flush();testEntityManager.clear();

    JPAQuery<Campaign> query = new JPAQuery(entityManager);
    List<Campaign> campaigns = query.select(qCampaign)
                                    .from(qCampaign)
                                    .join(qCampaign.events)
                                    .fetch(); // fetch시에 flush mode에 따라서 flush가 발생

    assertThat(campaigns).hasSize(1);
    assertThat(campaigns.get(0).getEvents()).hasSize(1);
    assertThat(campaigns.get(0).getEvents().get(0).getId()).isNotNull();
}
```

```sql
select
    campaign0_.id as id1_0_,
    campaign0_.amount as amount2_0_,
    campaign0_.name as name3_0_ 
from
    campaign campaign0_ 
inner join
    event events1_ 
on campaign0_.id=events1_.campaign_id
```

```sql
Hibernate: 
    select
        events0_.campaign_id as campaign4_1_0_,
        events0_.id as id1_1_0_,
        events0_.id as id1_1_1_,
        events0_.amount as amount2_1_1_,
        events0_.campaign_id as campaign4_1_1_,
        events0_.name as name3_1_1_ 
    from
        event events0_ 
    where
        events0_.campaign_id=?
```

- 일반 Join만 사용하는 경우 select 부분에 campaign 관련 내용만 존재하게 된다.
- 그리고 Campaign의 Event의 데이터를 조회할 시에 event를 다시 select하기에 N+1이 발생할 수 있다.

## FetchJoin

```java
@Test
void fetchJoinTest() {
    String name = RandomStringUtils.randomAlphabetic(5);
    long amount = RandomUtils.nextLong();
    Campaign campaign = generateCampaign(name, amount);
    testEntityManager.flush();testEntityManager.clear();

    JPAQuery<Campaign> query = new JPAQuery(entityManager);
    List<Campaign> campaigns = query.select(qCampaign)
                                    .from(qCampaign)
                                    .join(qCampaign.events).fetchJoin()
                                    .fetch(); // fetch시에 flush mode에 따라서 flush가 발생

    assertThat(campaigns).hasSize(1);
    assertThat(campaigns.get(0).getEvents()).hasSize(1);
    assertThat(campaigns.get(0).getEvents().get(0).getId()).isNotNull();
}
```

```sql
Hibernate: 
    select
        campaign0_.id as id1_0_0_,
        events1_.id as id1_1_1_,
        campaign0_.amount as amount2_0_0_,
        campaign0_.name as name3_0_0_,
        events1_.amount as amount2_1_1_,
        events1_.campaign_id as campaign4_1_1_,
        events1_.name as name3_1_1_,
        events1_.campaign_id as campaign4_1_0__,
        events1_.id as id1_1_0__ 
    from
        campaign campaign0_ 
    inner join
        event events1_ 
            on campaign0_.id=events1_.campaign_id
```

- fetchJoin을 사용하게 되면 select에는 campaign을 가져오라고 되어 있지만 실제로는 campaign과 campaign id로 되어 있는
event 들을 전부 가져오게 되어 있다.
- 따라서 campaign에 있는 event를 사용할 때 N+1 query가 발생하지 않는다.

## FetchJoin Use On

```java
@Test
void joinUseOnTest() {
    String name = RandomStringUtils.randomAlphabetic(5);
    long amount = RandomUtils.nextLong();
    Campaign campaign = generateCampaign(name, amount);
    testEntityManager.flush();testEntityManager.clear();

    JPAQuery<Campaign> query = new JPAQuery(entityManager);
    List<Campaign> campaigns = query.select(qCampaign)
                                    .from(qCampaign)
                                    .join(qCampaign.events).fetchJoin()
                                    .join(qCampaign.events, qEvent).on(qEvent.campaign.id.eq(campaign.getId()))
                                    .fetch(); // fetch시에 flush mode에 따라서 flush가 발생

    assertThat(campaigns).hasSize(1);
    assertThat(campaigns.get(0).getEvents()).hasSize(size);
    assertThat(campaigns.get(0).getEvents().get(0).getId()).isNotNull();
}
```

```sql
select
    campaign0_.id as id1_0_0_,
    events1_.id as id1_1_1_,
    campaign0_.amount as amount2_0_0_,
    campaign0_.name as name3_0_0_,
    events1_.amount as amount2_1_1_,
    events1_.campaign_id as campaign4_1_1_,
    events1_.name as name3_1_1_,
    events1_.id as id1_1_0__ 
    events1_.campaign_id as campaign4_1_0__,
from
    campaign campaign0_ 
    inner join
    event events1_ 
        on campaign0_.id=events1_.campaign_id 
inner join
    event events2_ 
        on campaign0_.id=events2_.campaign_id 
        and (
            events2_.campaign_id=?
              )
```

- Join에 On을 이용해서 조건을 붙이고 싶을 때는 어떻게 써야할까 고민을 할 수 있다.
- 조건을 사용할 때는 까다로워 지는게, On을 한 join에는 fetchJoin을 사용할 수 없다.
- 따라서 fetchJoin을 한 join과 On을 사용한 join을 함께 사용 해야 한다.

## FetchJoin Where Condition
```java
@Test
void joinWhereConditionTest() {
    String name = RandomStringUtils.randomAlphabetic(5);
    long amount = RandomUtils.nextLong();
    Campaign campaign = generateCampaign(name, amount);
    testEntityManager.flush();testEntityManager.clear();

    JPAQuery<Campaign> query = new JPAQuery(entityManager);
    List<Campaign> campaigns = query.select(qCampaign)
                                    .from(qCampaign)
                                    .join(qCampaign.events, qEvent).fetchJoin()
                                    .where(qEvent.campaign.id.isNull().or(qEvent.campaign.id.eq(campaign.getId())))
                                    .fetch(); // fetch시에 flush mode에 따라서 flush가 발생

    assertThat(campaigns).hasSize(1);
    assertThat(campaigns.get(0).getEvents()).hasSize(size);
    assertThat(campaigns.get(0).getEvents().get(0).getId()).isNotNull();
}
```

```sql
Hibernate: 
select
     campaign0_.id as id1_0_0_,
     events1_.id as id1_1_1_,
     campaign0_.amount as amount2_0_0_,
     campaign0_.name as name3_0_0_,
     events1_.amount as amount2_1_1_,
     events1_.campaign_id as campaign4_1_1_,
     events1_.name as name3_1_1_,
     events1_.campaign_id as campaign4_1_0__,
     events1_.id as id1_1_0__ 
from
     campaign campaign0_ 
inner join
     event events1_ 
         on campaign0_.id=events1_.campaign_id 
where
     events1_.campaign_id is null 
     or events1_.campaign_id=?
```

- join을 두번 하는게 이상하다면, Where를 사용해서 같은 결과를 얻을 수 있다.
- 단, where와 on을 사용할 때 질의 범위가 다르기에 원하는 결과가 나오는지 확인 하는 작업이 필요하다.

## Fake FetchJoin

```java
@Test
void fakeFetchJoinTest() {
    String name = RandomStringUtils.randomAlphabetic(5);
    long amount = RandomUtils.nextLong();
    Campaign campaign = generateCampaign(name, amount);
    testEntityManager.flush();testEntityManager.clear();

    JPAQuery<Campaign> query = new JPAQuery(entityManager);
    List<Campaign> campaigns = query.select(qCampaign)
                                        .from(qCampaign)
                                        .join(qEvent).on(qEvent.campaign.id.eq(campaign.getId())).fetchJoin()
                                        .fetch(); // fetch시에 flush mode에 따라서 flush가 발생

    assertThat(campaigns).hasSize(1);
    assertThat(campaigns.get(0).getEvents()).hasSize(size);
    assertThat(campaigns.get(0).getEvents().get(0).getId()).isNotNull();
}
```

```sql
Hibernate: 
select
    campaign0_.id as id1_0_,
    campaign0_.amount as amount2_0_,
    campaign0_.name as name3_0_ 
from
    campaign campaign0_ 
inner join
    event event1_ 
        on (
        event1_.campaign_id=campaign0_.id
        )
```

- 이것도 `Campaign`과 `Event`과 join을 하고 fetchJoin까지 한 것 처럼 보인다.
- Query를 자세히 보면, select에 campaign만 존재한다.
- 그래도 신기하게 join에 조건은 원하는 조건을 만족하고 있어서 결과값은 정상으로 나오게 되어 있다.
- 그렇지만 campaign의 event를 사용하는 순간 N+1이 발생할 것이다.
- Select되는 QModel에 연관되 있는 데이터를 사용해야 하는데, db 쿼리 관점으로 생각해서 작성하게 되면
이러한 실수를 할 수 있으니 조심해야 한다.

## 마치며
`fetchJoin`을 쓸 때 왜 QModel에 있는 모델을 join에 넣었는지 이해하지 못했었는데,
다 이유가 있었던 것이다...

db 쿼리 관점에서 코드를 작성해도 동작은 하나, 다른 이슈가 발생하니 잘 써야 좋은데 모르고 쓰면 성능에 더 악영향을 미칠 수 있다는 
것을 깨달았다.

그리고 fetchJoin을 안써도 같이 조회가 되는 케이스가 있는데, select에 join된 테이블의 값을 사용하는 경우 fetchJoin을 안하더라도
한번에 조회가 되게 되어 있다.

join을 쓰게되면 중복이 발생할 수 있기에 `distinct`를 잘 써주어야 한다!
[goto](https://dotoridev.tistory.com/2?category=0)
- - - 
[fetchJoinTest](https://github.com/KangWooJin/spring-study/blob/master/querydsl/src/test/java/kangwoojin/github/io/querydsl/QuerydslFetchJoinTest.java)
관련 example code는 github에 올려두었으니 참고~!
 