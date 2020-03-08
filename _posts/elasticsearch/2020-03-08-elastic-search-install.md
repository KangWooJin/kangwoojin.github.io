---
title: "[ElasticSearch] Elastic search, AWS EC2 셋팅하기"
categories:
  - Programing
tags:
  - Elasticsearch
toc: true
toc_sticky: true
date: 2020-03-08 22:00:00+09:00 
excerpt: AWS EC2에 elastic search를 설치해서 확인하기
---

## 들어가며
AWS EC2에 elastic search를 설치해 보자~

elastic search에 대해서 간단하게 공부하기 위해서 서버를 직접 구축해보고 
테스트 해보기 위해 시작하였습니다.

 
## 개발 환경

- AWS EC2 - 프론티어(Amazon Linux 2 AMI (HVM), SSD Volume Type)
- Elastic search - 7.6.1

## 설치하기

elastic search에서 제공하는 install 방식을 참고하였습니다.

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.6.1-linux-x86_64.tar.gz
tar -xzf elasticsearch-7.6.1-linux-x86_64.tar.gz
cd elasticsearch-7.6.1/ 
```

[https://www.elastic.co/guide/en/elasticsearch/reference/7.6/targz.html](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/targz.html)

## 셋팅
EC2 서버 답게 메모리가 엄청 작게 셋팅 되어 있습니다.
사용 가능한 메모가 총 983M으로 엄청 작게 되어 있어 elastic search 기본 설정을 변경해야 합니다.
```bash
              total        used        free      shared  buff/cache   available
Mem:           983M        584M        141M        400K        257M        264M
Swap:            0B          0B          0B
``` 

간단하게 데이터 몇 개만 넣어보고 index 생성, alias 변경 정도로만 할 예정이기에
300M정도 잡도록 하겠습니다.

설치된 `$ES_HOME/config`에 `jvm.options`에 접근하여 `-Xms`와 `-Xmx`를 변경합니다.
```bash
# Xms represents the initial size of total heap space
# Xmx represents the maximum size of total heap space

-Xms300m
-Xmx300m
```

이 상태로 실행을 하게 되면 성공할 수 있지만 validation에 의해서 실패하게 됩니다.

```bash
./bin/elasticsearch -d -p pid
```

## 에러 수정

아래와 같은 에러가 발생하게 되면 리눅스 기본 커널 설정을 변경해줘야 합니다.

```bash
[4] bootstrap checks failed
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]
[2]: max number of threads [3792] for user [ec2-user] is too low, increase to at least [4096]
[3]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[4]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
```

### max file descriptors 변경

```bash
sudo vi /etc/security/limits.conf

ec2-user hard nofile 65536 # ec2-user는 하드세팅(해당 쉘 최대값) 으로 한 번 접속할때 65536번의 파일을 열어 볼 수 있습니다.
ec2-user hard nofile 65536 # ec2-user는 소프트세팅(해당 설정 값) 으로 한 번 접속할때 65536번의 파일을 열어 볼 수 있습니다.
ec2-user hard nproc 65536 # ec2-user는 하드세팅으로 한 번 접속할때 65536번의 프로시저를 실행 할 수 있습니다. 
ec2-user sort nproc 65536 # ec2-user는 소프트세팅으로 한 번 접속할때 65536번의 프로시저를 실행 할 수 있습니다.
```

 
### max virtual memory 변경
해당 방법은 root 권한을 갖고 있어야 합니다.
만약 root 계정이 셋팅 안되어 있다면 패스워드 설정 후 su 명령어를 통해서 root로 로그인 해야 됩니다.

```bash
su passwd root
``` 

vi로 열어서 수정해보려고 했지만 잘 안되서 echo로 덮어 썼습니당 ㅠㅠ

```bash
echo 262144 > /proc/sys/vm/max_map_count
```

### discovery와 master node 셋팅
`elasticsearch.yml`에 있는 discovery에 값을 변경해주어야 합니다.

`discovery.seed_hosts`와 `cluster.initial_master_nodes`의 값을 셋팅 해줘야 합니다.

default로 셋팅이 될거라 예상했지만 값을 입력을 해줘야 해서 local ip를 셋팅해주었습니다.


```bash
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
discovery.seed_hosts: ["127.0.0.1", "[::1]"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
#
cluster.initial_master_nodes: ["127.0.0.1"] # 해당 서버의 ip 주소 입력
#
# For more information, consult the discovery and cluster formation module documentation.
#
```

참고
- [http://libqa.com/wiki/807](http://libqa.com/wiki/807)
- [https://kugancity.tistory.com/entry/elasticsearch-632-설치하기](https://kugancity.tistory.com/entry/elasticsearch-632-%EC%84%A4%EC%B9%98%ED%95%98%EA%B8%B0)

## 확인
내부에서 확인

```bash
curl localhost:9200
```

```bash
{
  "name" : "first",
  "cluster_name" : "elastic-7.6.1",
  "cluster_uuid" : "lBpfYNxZT0iyZxwfC8uxlA",
  "version" : {
    "number" : "7.6.1",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "aa751e09be0a5072e8570670309b1f12348f023b",
    "build_date" : "2020-02-29T00:15:25.529771Z",
    "build_snapshot" : false,
    "lucene_version" : "8.4.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

최종 목적은 외부에서 해당 es 서버를 사용하는게 목적이기에 해당 포트를 외부에서 접근 가능하게 EC2에서 오픈 해줘야 합니다.

1. 해당 인스턴스에 보안 그룹에 접근
2. Inbound rules에 Custom TCP, 9200 포트를 자신의 IP 또는 모든 ip에 대해서 오픈

오픈 후 테스트 진행
```bash
curl 외부ip:9200

curl: (7) Failed to connect to xxx.amazonaws.com port 9200: Connection refused
```

elastic search의 기본 network는 localhost로 접근되기 때문에 외부에서 접근이 불가능합니다.
따라서 elastic search의 network config를 수정해줘야 합니다.

```bash
# ---------------------------------- Network -----------------------------------
#
# Set the bind address to a specific IP (IPv4 or IPv6):
#
network.host: xxx.xxx.xxx.xxx # 자신의 ip 입력
#
# Set a custom port for HTTP:
#
http.port: 9200
#
# For more information, consult the network module documentation.
```

해당 ip로 입력 후 서버에서 curl을 통해서 다시 확인을 해보면 잘 나오는 것을 확인할 수 있습니다.

* elastic을 간단하게 start, stop하기 위한 script

```bash
#!/bin/bash
HOME=/home/ec2-user/app/elasticsearch-7.6.1
ARGS=$1

if [ $# -ne 1 ]; then
	echo "script need 1 paramater"
	exit 1
fi

case $ARGS in
	start)
		echo "es start"
		$HOME/bin/elasticsearch -d -p pid
		;;
	stop)
		echo "es stop"
		pkill -F $HOME/pid
		;;
	*)
		echo "Sorry, I don't understand"
		;;
esac
```

## 마치며

올해의 목표 중 나만의 서버를 갖게 하는 것도 있고 elastic search에 대해서도 공부해보고 싶은 부분이 있어
간단하게 셋팅을 해보았는데 해당 서버에 spring data elastic을 적용해보고 mapping에 대해 공부해볼 예정입니다.

간단하게 사용하는데 비용이 많이 나오면 EC2 말고 다른 걸로 갈아타야 겠다 ㅠ 