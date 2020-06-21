---
title: "[Spring] @Transactional propagation 동작 방식 기초" 
categories:
  - Programing
tags:
  - Spring
toc: true
toc_sticky: true
date: 2020-06-21 08:00:00+09:00 
excerpt: Spring에서 Transaction을 사용할 때 propagation이 어떻게 전파되는지 알아보자.
---

## 들어가며
@Transactional propagation에 대해서 어떻게 동작하는지 확인 해보고 싶어서 작성한다.

## 테스트 조건
- Campaign - Event 는 OneToOne 관계이다.
- Campaign을 등록 할때는 Event의 값이 필요조건이다.

```java
@Entity
@Data
public class Campaign {
    @Id
    @GeneratedValue
    private Long id;

    private String name;

    @NotNull
    @OneToOne
    private Event event;
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
}
```

## TestCode 

- CampaignController
- Event를 먼저 생성 후 Campaign을 생성 한다.

```java
@RequiredArgsConstructor
@RestController
public class CampaignController {
    private final CampaignService campaignService;
    private final EventService eventService;

    @PostMapping("/create")
    @Transactional
    public Campaign create(@RequestParam String name) {
        Event event = new Event();
        event.setName(name);
        return campaignService.process(eventService.save(event));
    }
}
```

- EventService
- JpaRepository를 이용해서 간단하게 save 한다.

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class EventService {
    private final EventRepository eventRepository;

    @Transactional
    public Event save(Event event) {
        return eventRepository.save(event);
    }
}
```

- CampaignService
- process method에서 Event를 전달 받아 Campaign을 생성한다.

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class CampaignService {
    private final CampaignRepository campaignRepository;

    @Transactional
    public Campaign save(Campaign campaign) {
        if (campaign.getEvent() != null) {
//            throw new RuntimeException("runtimeException");
        }
        return campaignRepository.save(campaign);
    }

    @Transactional
    public Campaign process(Event event) {
        Campaign campaign = new Campaign();
        campaign.setEvent(event);
        campaign.setName("campaign");

        return save(campaign);
    }
}
```

- mvc mock test
 
```java
@AutoConfigureMockMvc
@SpringBootTest
public class CampaignControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @Test
    void createTest() throws Exception {
        MvcResult mvcResult = mockMvc.perform(post("/create")
                                                      .param("name", "event")
                                                      .contentType(MediaType.APPLICATION_JSON))
                                     .andReturn();
    }
}

```

## Transaction 분석

- log level을 Trace로 하여 추적해보면서 확인한 부분에 대해서 정리한다.
- log data가 너무 길고, 보기 어려워 필요한 부분만 남기고 삭제하였다.
- 중요한 부분은 `TransactionManager`에서 트랜잭션을 create, commit, rollback을 관리하게 된다.

```
Creating new transaction with name [.controller.CampaignController.create]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
begin
Initializing transaction synchronization
Getting transaction for [.controller.CampaignController.create]
Participating in existing transaction
Getting transaction for [.service.EventService.save]
Participating in existing transaction
Getting transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
Completing transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
Completing transaction for [.service.EventService.save]
Participating in existing transaction
Getting transaction for [.service.CampaignService.process]
Participating in existing transaction
Getting transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
Completing transaction for [org.springframework.data.jpa.repository.support.SimpleJpaRepository.save]
Completing transaction for [.service.CampaignService.process]
Completing transaction for [.controller.CampaignController.create]
Triggering beforeCommit synchronization
Triggering beforeCompletion synchronization
Initiating transaction commit
committing
Triggering afterCommit synchronization
Clearing transaction synchronization
Triggering afterCompletion synchronization
``` 

- 최초로 만나는 `@Transactional` annotation에서 transcation이 생성된다.
- 그 후 만나는 `@Transactional`은 propagation DEFAULT 설정인 `REQUIRED`에 의해서 상위 transactioon을 재사용 한다.
- 같은 class에 있는 method에 둘다 `@Transactional`이 존재하는 경우 proxy를 이용하기에 external 요청으로 온 최초 method에서만 동작하게 된다.
(트랜잭션을 사용할 때 주의해야 하는 부분이다.)
- Transaction이 Complete 될때는 stack 방식으로 동작한다.
- commit 관련된 hook이 4가지가 존재 한다.
- 순서는 beforeCommit -> beforeCompletion -> commit -> afterCommit -> afterCompletion  
- 확인하다보니 SimpleJpaRepository에서 save 시점에도 `@Transcational`이 존재 하였다.

## 마치며
정말 간단하게 `Transcation`이 어떤식으로 진행되는지 알아보았다.

원래는 propagation을 서로 다르게 적용해보면서 테스트를 해보려 하였는데 삽질을 많이 하여 시간을 많이 사용하게 되었다...
(어떤식으로 동작되는지 궁금할때는 log level을 trace로 바꿔서 확인하기..)

`propagation`, `isolation`에 대해서 서로 다르게 적용했을 때 어떻게 되는지는 다음에 정리하기로 한다.    

참고 : [spring transcation manager](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/data-access.html#transaction-declarative-annotations)
- - - 
[transaction](https://github.com/KangWooJin/spring-study/commit/a7d0d34ddd809a4965ceae6976af93b019c19ba7)
관련 example code는 github에 올려두었으니 참고~!
 