---
title: "[Querydsl] Querydsl을 사용하면서 실수하기 쉬운 부분 정리" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-07-05 08:00:00+09:00 
excerpt: Querydsl을 사용할 때 실수하기 쉬운 부분에 대해서 알아보고 제대로된 사용 방법에 대해 알아두자.
---

## 들어가며
Querydsl을 사용할 때 실수 했던 부분에 대해서 기록해두고 다음에는 실수하지 않기 위한 목적으로
글을 정리한다.

## select 주의사항
- querydsl에서는 QModel을 사용하여 select할 데이터를 지정하게 되어 있다.
- 기본적으로 전체조회를 하는 경우 QModel을 그대로 사용하여 select 하는게 일반적이다.
- 필요한 데이터만 읽기 위해서는 QModel에 필드를 select에 작성해 주어야 한다.
- 필요한 필드만 작성하게되면 querydsl에서는 `Tuple`로 반환하게 되고, `Tuple`에서 key 기반으로 데이터를 꺼내서 사용해야 한다.
- `Tuple`을 꺼낼 때 select에 사용된 QModel에 key로 값이 들어가게 되어 있으므로 주의가 필요하다.

### qModel에 field를 사용하는 경우

```java
@Override
public Tuple selectUseQModelFields(Long id) {
  return from(qCampaign)
         .select(qCampaign.id, qCampaign.name, qCampaign.amount)
         .where(qCampaign.id.eq(id))
         .fetchOne();
}
```

- QModel에 있는 field를 select할 수 있게 조건을 추가한다.

```java
@Test
void selectUseQModelFields() {
  long campaignAmount = 5L;
  Campaign campaign = new Campaign();
  campaign.setAmount(campaignAmount);
  String campaignName = "test";
  campaign.setName(campaignName);

  Campaign saved = testEntityManager.persist(campaign);
  testEntityManager.flush(); testEntityManager.clear();

  Tuple tuple = campaignRepository.selectUseQModelFields(saved.getId());

  assertThat(tuple.get(qCampaign.id)).isEqualTo(saved.getId());
  assertThat(tuple.get(qCampaign.name)).isEqualTo(saved.getName());
  assertThat(tuple.get(qCampaign.amount)).isEqualTo(saved.getAmount());
}
```

- 사용하는 곳에서는 반드시 QModel에 field를 `tuple.get()`에 key로 사용해야 원하는 값을 읽을 수 있다.


### QModel 그 자체로 사용 하는 경우

```java
@Override
public Tuple selectUseQModelOnly(Long id) {
  return from(qCampaign)
               .select(qCampaign, qCampaign.amount.add(1L))
               .where(qCampaign.id.eq(id))
               .fetchOne();
}
```

- QModel 자체를 select에 넣고 조회하는 경우 기본적으로 QModel의 domain class를 반환하기에
Tuple을 반환할 수 있게 단일 조회 부분을 추가하였다.

```java
@Test
void selectUseQModelOnly() {
  long campaignAmount = 5L;
  Campaign campaign = new Campaign();
  campaign.setAmount(campaignAmount);
  String campaignName = "test";
  campaign.setName(campaignName);

  Campaign saved = testEntityManager.persist(campaign);
  testEntityManager.flush(); testEntityManager.clear();

  Tuple tuple = campaignRepository.selectUseQModelOnly(saved.getId());

  assertThat(tuple.get(qCampaign.id)).isNull();
  assertThat(tuple.get(qCampaign.name)).isNull();
  assertThat(tuple.get(qCampaign.amount)).isNull();
  assertThat(tuple.get(qCampaign).getId()).isEqualTo(saved.getId());
  assertThat(tuple.get(qCampaign).getName()).isEqualTo(saved.getName());
  assertThat(tuple.get(qCampaign).getAmount()).isEqualTo(saved.getAmount());
  assertThat(tuple.get(qCampaign.amount.add(1L))).isEqualTo(saved.getAmount()+1);
}
```

- QModel에 필드 값으로 조회한 부분은 `Null`로 나오고 `tuple.get(QCampaign).getXXX()`
로 한 데이터들에만 값이 존재하였다.
- 해당 부분은 컴파일 에러도 생기지 않아 런타임시에 데이터가 없어 `NullPointException`이 발생하면
에러 파악을 할 수 있지만, 그렇지 않은 경우 확인하기가 어렵다.
- 따라서 select하고 사용할 때 주의가 필요하다.

## leftJoin에 groupBy 사용시 aggregation 값이 Null
- aggregation을 하게 되면 보통 값이 없으면 0을 반환할거라 예상되지만, 실제로 0이 아닌 `null`을 반환하기에
주의가 필요하다.

```java
@Override
public List<CampaignDto> getAmountSum(Long eventId) {
  return from(qCampaign)
               .select(qCampaign.id, qCampaign.name, qCampaign.amount.sum(), qEvent.amount.sum())
               .distinct()
               .leftJoin(qCampaign.events, qEvent).on(qEvent.id.eq(eventId))
               .groupBy(qCampaign.name)
               .fetch()
               .stream()
               .peek(tuple -> log.info("tuple {}", tuple))                
               .map(tuple -> new CampaignDto().setCampaignId(tuple.get(qCampaign.id))
                                              .setCampaignName(tuple.get(qCampaign.name))
                                              .setAmountSum(tuple.get(qCampaign.amount.sum()))
                                              .setEventAmountSum(tuple.get(qEvent.amount.sum())))
               .collect(Collectors.toList());

}
```

- `QCampaign`에 `amonut.sum`과 `QEvent`의 `amount.sum`을 할 때 `QEvent`쪽 데이터가 없는 경우
`qEvent.amount.sum()`의 값은 null로 return 된다.

```java
@Test
void groupByLeftJoinIsEmptyThenAggregationValueIsNullTest() {
  Campaign campaign = new Campaign();
  long campaignAmount = 5L;
  String campaignName = "test";
  campaign.setAmount(campaignAmount);
  campaign.setName(campaignName);
  testEntityManager.persist(campaign);
  testEntityManager.flush(); testEntityManager.clear();

  List<CampaignDto> actual = campaignRepository.getAmountSum(0L);

  assertThat(actual).isNotEmpty();
  assertThat(actual).hasSize(1);
  assertThat(actual.get(0).getCampaignName()).isEqualTo(campaignName);
  assertThat(actual.get(0).getAmountSum()).isEqualTo(campaignAmount);
  assertThat(actual.get(0).getEventAmountSum()).isNull();
}
```

- leftJoin을 사용할 때 join된 데이터가 없을 수 있다는 가정하에 하기에, Null 값을 반환할 수 있다는
생각을 반드시 해야 한다.

## 마치며
- querydsl을 사용할 때 자연스럽게 실수를 할 수 있는 부분에 대해서 정리를 해보았다.
- 해당 두 가지는 실제로 겪어봤기에... querydsl을 사용할 때 주의 또 주의하려고 한다!
 

- - - 
[querydslMistake](https://github.com/KangWooJin/spring-study/commit/facf34cc367bdbb67f90e036fd355e03e44a21f0)
관련 example code는 github에 올려두었으니 참고~!
 