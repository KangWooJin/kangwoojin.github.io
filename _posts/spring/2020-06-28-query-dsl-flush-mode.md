---
title: "[Querydsl] Querydsl flush mode" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-06-28 08:00:00+09:00 
excerpt: Querydsl flush mode에 대해서 알아보자.
---

## 들어가며
Querydsl에서 JPQL을 실행할 때 flush mode에 따라 동작되는 방식을 알아보자.

## JPA와 Flush

- JPA에서는 Query에 대해서 지연쓰기를 지원하기에 insert, update, delete를
하더라도 flush가 발생하기 전까지는 db에 반영이 되지 않는다.
- 그렇기에 강제로 flush를 하던지, commit을 해주어야 한다.

## FlushMode
- `FlushModeType`는 `AUTO`, `COMMIT` 두 가지로 되어 있는 `enum`이다.
- `default`는 `AUTO`이며, AUTO flush가 발생하는 조건은 총 3가지 경우가 존재 한다.
- `hibernate`에서는 더 다양한 상태가 존재하지만 query dsl에서는 2가지만 존재 한다.

1. transaction이 commit이 되기전
2. JPQL/HQL의 쿼리 실행 전
3. native SQL이 실행되기 전

### AutoFlush

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

- `Campaign`과 `Event`는 1:N이며, 양방향 연관관계를 설정하였다.


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

- `generatedCampaign`시에 `campaign`을 insert 후, event에 campaign을 할당 해주었다.
- `Campaign`에 cascade가 `PERSIST`이기에 campaign이 생성되면 campaign에 등록된 event들도 전부 insert하게 된다.
- event에 campaign의 값을 update하여 event의 상태는 drity check 상태이다.
- 그렇지만 JPA에서는 쓰기지연이 발생하기에 flush가 발생하기 전에는 db에는 아무런 변화가 없다.
 
```java
@Test
void flushModeWhenFlushModeIsAutoTest() {
  String name = RandomStringUtils.randomAlphabetic(5);
  long amount = RandomUtils.nextLong();
  Campaign campaign = generateCampaign(name, amount);

  JPAQuery<Campaign> query = new JPAQuery(entityManager);
  List<Campaign> campaigns = query.select(qCampaign)
                                  .from(qCampaign)
                                  .join(qCampaign.events)
                                  .setFlushMode(FlushModeType.AUTO) // default
                                  .fetch(); // fetch시에 flush mode에 따라서 flush가 발생

  assertThat(campaigns).hasSize(1);
}
```

- 테스트를 진행했을 때 fetch가 발생하게 되면 설정된 `FlushMode`에 따라 동작 방식이 바뀐다.
- AUTO로 진행하였기에 앞에 말했던 3가지 조건 중 JPQL이 발생하여 auto flush가 되고 select가 동작하기에
해당 테스트에 문제가 되지 않는다.

```java
@Test
void flushModeWhenFlushModeIsCommitTest() {
    String name = RandomStringUtils.randomAlphabetic(5);
    long amount = RandomUtils.nextLong();
    Campaign campaign = generateCampaign(name, amount);

    JPAQuery<Campaign> query = new JPAQuery(entityManager);
    List<Campaign> campaigns = query.select(qCampaign)
                                    .from(qCampaign)
                                    .join(qCampaign.events)
                                    .setFlushMode(FlushModeType.COMMIT)
                                    .fetch(); // fetch시에 flush mode에 따라서 flush가 발생

    assertThat(campaigns).hasSize(0);
}
```

- `FlushMode`를 `COMMIT`으로 변경하여 테스트를 하는 경우 campaign이 조회가 되지 않는다.
- 이유는 JPQL의 경우 영속성 컨텍스트에서 조회를 하지 않고 실제 db에서 조회를 하기 때문이다.
- flush가 발생하지 않았기에 영속성 컨텍스트에만 campaign, event의 값이 셋팅이 되어 있다.
- 참고로 commit은 트랜잭션이 종료 되기 전에 발생하거나, 강제로 발생시켜야 한다.

## 마치며
JPA 관련 책을 읽고 정리했을 때 해당 내용이 있었는데, 어느 순간 까먹게 되었다.

아마 책을 읽는 순간과 정리를 할 때 코드를 직접 실행 해보지 않았기에 기억이 오래가지 않는 것 같다.

사람마다 학습하는 타입이 다르다고 들었는데, 책을 읽고도 쉽게 이해가 되는 타입과 해보면서 이해하는 타입 중
나는 해봐야 이해가 되는 타입 인거 같다...

- - - 
[flushModeTest](https://github.com/KangWooJin/spring-study/blob/master/querydsl/src/test/java/kangwoojin/github/io/querydsl/FlushModeTest.java)
관련 example code는 github에 올려두었으니 참고~!
 