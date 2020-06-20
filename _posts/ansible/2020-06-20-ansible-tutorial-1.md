---
title: "[Ansible] Ansible tutorial(1) - getting started ansible" 
categories:
  - Programing
tags:
  - Ansible
toc: true
toc_sticky: true
date: 2020-06-20 11:00:00+09:00 
excerpt: Ansible을 이용해서 환경설정 관리를 편리하게 해보자.
---

## 들어가며
ansible을 왜 써야할까를 고민 한다면 자신의 조직이 어떤 상황 인지 파악한 뒤에 적용 하는게 좋아 보인다.

그래서 ansible을 왜 쓰게 되 었는지 부터 적어 본다.

### 중복 되는 환경 설정 관리

```
// A component
env=Foo
start A

// B component
env=Foo
start B
```

- 해당 스크립트를 관리한다고 가정했을 때 env 설정은 같지만 component가 다르기에 따로 관리하고 있다.
- 따라서 추가적으로 `env=Foo,Bar`로 전체적으로 수정하고 싶은 경우 component 마다 스크립트를 수정해줘야 한다.
- component의 갯수가 1~2개면 손으로하지 하겠지만 10개, 100개가 넘어간다면 멘붕이 찾아올 수 있다.
- ansible을 이용하게 되면 아래의 스크립트 한개로 컴포넌트 별 스크립트를 관리할 수 있다.

```
env=Foo,Bar
start {{ component }}
``` 

## 용어 정리
### Control Node
- Ansible을 실행하는 서버를 의미한다.
- control Node는 managed node와 ssh 통신을 할 수 있어야 한다.

### Managed node
- control Node에 의해 관리되어지는 서버들을 의미한다.

### Host inventory
- managed node의 서버 리스트를 의미하고, inventory 라고도 부른다.

### Ad-hoc command
- 간단한 일회성 작업

### Playbook
- 반복적인 작업들을 설정 해놓은 환경설정들이다.

### Module
- 사용자 추가, 패키지 설치 등과 같은 특정 공통 작업을 수행하는 코드

### Idempotency
- 멱등성을 보장 한다.
- 한 번 실행한 결과가 어떠한 개입도 하지 않고 반복적으로 수행한 결과와 정확히 같을 경우 연산이 무효가 된다.

## 설치
```
sudo yum install -y ansible
```
- 대부분 리눅스 서버를 사용하니 yum을 이용해 설치하는게 간단하다.
- 혹시나 나중에 필요할 수도 있으니 ssh 관련 스크립트를 추가한다.

### deploy key script  

```
#!/bin/bash

if [[ "$#" -lt 1 ]]; then
  echo "usage: deploy_key.sh key_file_name"
  exit 1
fi

setup_log="\e[32m[Preparation]\[0m"

base_dir=$(
  cd $(dirname $0)
  pwd
)
cd ${base_dir}

user=$(whoami)
key_file_name=$1

key=$(cat ${key_file_name})
echo -e "$setup_log Key: $key"

function deploy_key() {
  local home_dir=$1
  cd ${home_dir}

  if [[ ! -d .ssh ]]; then
    mkdir -p .ssh
    chmod 700 .ssh
  fi

  if [[ ! -f .ssh/authorized_keys ]]; then
    echo "$key" >.ssh/authorized_keys
    chmod 600 .ssh/authorized_keys
  elif ! grep -qs "${key}" .ssh/authorized_keys; then
    echo "$key" >>.ssh/authorized_keys
    echo "key appended"
  else
    echo "key already exists"
  fi
}

home_dir="/$user"
echo -e "$setup_log Home Dir: ${home_dir}"

deploy_key ${home_dir}
```

## 설정
- ansible에도 config가 존재 하는데 설치가 완료되면 `/etc/ansible/`를 global 값들이 설정된다.
- `ansible.cfg`가 config 파일 이름이며, 우선순위가 존재한다.
```
가장 위에 있는걸로 적용 된다.
global -> home directory -> per directory -> environment variable
1. ANSIBLE_CONFIG (environment variable)
2. ansible.cfg (per directory)
3. ~/.ansible.cfg (home directory)
4. /etc/ansible/ansible.cfg (global)
``` 

## host inventory
- managed host들을 관리하기 위해 `hosts` 라는 파일로 서버들을 적어둔다.
- 기본적으로 `/etc/ansible/hosts`의 값을 이용하지만 `-i` 옵션을 사용하여 변경할 수 있다.
- `ansible-playbook -i release test.yml`
- 정적으로 대부분 사용하지만 동적으로도 hosts를 적용할 수 있다.

```
[webservers]
vm2
vm3
# vm[2:3] range도 가능
[dbservers]
vm4

[logservers]
vm5

# lamp라는 name으로 webservers, dbservers에 설정된 hosts를 셋팅 한다.
[lamp:children]
webservers
dbservers
```

- 설정된 inventory의 hosts를 확인하기

```
# global
ansible all --list-hosts

# custom inventory
ansible -i release all --list-hosts

# group
ansible logservers all --list-hosts
```

- 설정된 inventory에게 ping test 하기

```
ansible -i beta all -m ping
```

## Playbooks
- 각 서버들에게 설정되어야 하는 설정 값들을 `yaml` 파일로 관리할 수 있다.
- 아래와 같은 일을 n대의 서버에 반복적으로 해야 되는 경우 노가다를 할 순 없으니 Playbook 을 활용한다.

1. 해당 디렉토리가 있는지 확인한다.
2. 해당 디렉토리가 없으면 생성한다.
3. 해당 디렉토리에 특정 파일을 copy 한다.

```yaml
---
- hosts: logservers
 vars: 
   pinpoint_config_root: "/conf/pinpoint"
 tasks:
    - name: check pinpoint conf directory
      stat:
        path: "{{ pinpoint_config_root }}"
      register: pinpoint_config_dir
    
    - name: create pinpoint conf directory
      file:
        path: "{{ pinpoint_config_root }}"
        state: directory
      when: pinpoint_config_dir.stat.exists == False
    
    - name: Copy pinpoint conf
      template:
        src: "{{ item }}.jinja2"
        dest: "{{ pinpoint_config_root }}/{{ item }}"
        mode: 0664
      with_items:
        - pinpoint-env.config
```

- `copy-directory.yml`이라 가정 한다.
- `hosts`에 실행 시키고 싶은 inventory의 group name을 작성한다.
- `tasks` 하위에 앞에서 설정한 task를 설정한다. 
- `vars`를 이용해서 변수를 활용할 수 있다.
- `jinja2` 파일을 이용해서 해당 파일에도 `vars`나 global vars의 값을 주입해서 사용 가능하다.
- `ansible-playbook copy-directory.yml`
 
## 마치며
**한 두번의 설정은 손으로 하면 되지** 라고 생각 했었는데, 이거를 다음에 또 할려고 하니 사람이 할게 아닌거 같았다.

머리가 나쁘니 몸이 힘들어서, 몸을 편하게 하기 위해서 하나씩 쉽게 하는 법을 찾아보고 공부하는 것 같다.