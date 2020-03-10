---
title: "[ElasticSearch] Elastic search, kibana docker로 셋팅하기"
categories:
  - Programing
tags:
  - Elasticsearch
toc: true
toc_sticky: true
date: 2020-03-10 22:00:00+09:00 
excerpt: docker를 이용해 elasticsaerch, kibana를 셋팅 해보자~
---

## 들어가며
docker를 이용해 elasticsaerch, kibana를 셋팅 해보자~

AWS를 이용하니 ec2 서버가 가용중인 경우 시간에 비례해서 금액이 증가하여
간단하게 테스트할 용도다 보니 docker를 이용해서 로컬에서 띄우고 테스트 하는걸로 변경 하려고 합니다. 

 
## 개발 환경

- Docker - 19.03.5
- Elastic search - 7.6.1
- Kibana - 7.6.1

## 셋팅 하기

기본적으로 docker가 설치되어 있는 환경이라 가정하고 진행하였습니다.

`docker-compose.yml`을 이용해서 kibana와 elasticsearch를 한번에 관리하고
docker network를 같은 곳으로 셋팅하여 서로 통신이 가능하도록 합니다.

```yaml
version: '3'
services:
  kibana:
    image: kibana:7.6.1
    environment:
      SERVER_NAME: kibana
    ports:
      - 5601:5601
  elasticsearch:
    image: elasticsearch:7.6.1
    environment:
      - node.name=elasticsearch
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - discovery.type=single-node
    ports:
      - 9200:9200
      - 9300:9300
```

- [https://www.elastic.co/guide/en/elasticsearch/reference/7.6/docker.html](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/docker.html)
- [https://www.elastic.co/guide/en/kibana/current/docker.html](https://www.elastic.co/guide/en/kibana/current/docker.html)

yaml에 작성된 내용을 좀 더 자세하게 알아봅시다.

### version
- docker compose의 파일 규격을 정의합니다. 현재 '3'은 Swarm mode를 의미하며 network 설정이 포함되어 있습니다.
- 자세한 내용은 [https://docs.docker.com/compose/compose-file/compose-versioning/](https://docs.docker.com/compose/compose-file/compose-versioning/) 여기서 확인 가능합니다. 

### services
- 실행하고 싶은 이미지들을 리스트로 작성하여 병렬로 실행 시킵니다.
- `kibana`, `elasticsearch` 처럼 컨테이너의 이름들을 주고 나머지 값들을 셋팅 합니다.

### image
- 해당 컨테이너의 다운로드 받을 {container:tag} 작성하여 실행시에 로컬에 존재하지 않는 경우 다운로드하며 아닌 경우 존재하는 것을 사용합니다.

### environment
- 해당 이미지에서 사용되는 `env` 관련 설정을 맵 형태로 작성합니다.
- kibana에서도 elasticsearch 처럼 작성해도 괜찮으나 주의해야 할 점은 **대문자로만 작성해야 인식**하니 주의가 필요합니다.

### ports
- 해당 컨테이너의 포트와 로컬 서버의 포트를 연결해줍니다.
- 해당 ports를 작성하지 않은 경우 로컬에서 접근이 불가능하게 되어 있습니다.

## 마치며
`docker run`을 이용해서 하나씩 띄워보는 작업도 진행해봤지만 도커를 아직 잘 모르다보니
network 설정에 어려움이 있어서 docker compose의 version 3을 이용하면 
같은 network로 묶인 다고 되어 있어서 쉽게 셋팅할 수 있었습니다.

~~몇 시간을 삽질한 건 비밀😭~~

아쉬운거는 AWS의 켜둔 시간 만큼 과금이 붙는게 ㅠㅠ. 다른 클라우드 서비스를 찾아봐야 겠네요

- - - 

docker-compose는 [github](https://github.com/KangWooJin/spring-study/blob/master/elasticsearch/docker-compose.yml)에 올려두었습니다! 