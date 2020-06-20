---
title: "[Ansible] Ansible tutorial(2) - Playbook" 
categories:
  - Programing
tags:
  - Ansible
toc: true
toc_sticky: true
date: 2020-06-20 12:00:00+09:00 
excerpt: Ansible playbook을 이용해서 자주 활용 되는 환경을 모듈화 해보자.
jinja2_msg: "{% ... %}"
jinja2_msg2: "{{ ... }}"
jinja2_msg3: "{{ title }}"
jinja2_msg4: "{% ... %}"

---

## 들어가며
ansible-playbook을 사용하여 여러 서버에 작업해야 하는 부분을 하나의 스크립트를 통해서 쉽게 해결할 수 있었다.

그렇지만 각 서버마다 설정 값도 다르고, 서로 다른 yaml에서 공통되는 부분도 있고 다른 부분도 있을 때 
Roles를 통해서 모듈화하여 여러 yaml에서 사용할 수 있도록 알아보자.

## Roles
- ansible과 ansible-playbook은 차이가 있다.
- ansible은 바로 명령을 실행한다는 것과 ansible-playbook은 yaml을 실행 한다.
- 따라서 yaml에 자신이 실행하고자 하는 프로세스를 작성한다고 생각하면 된다.

### Directory structure

```yaml
a.yml
b.yml
roles/
    pinpoint-config/
        tasks/
        handlers/
        files/
        templates/
        vars/
        defaults/
        meta/
```
- pinpoint-config라는 설정이 a.yaml, b.yaml에서 사용된다고 가정해보자.
- ansible-playbook을 실행하는 폴더에서 `roles/pinpoint-config/tasks/main.yml`에 설정된 tasks를 읽어서 적용해 준다.
- 여러 yaml에서 사용되는 설정들을 role로 관리하게 되면 여러 yaml에서 쉽게 사용할 수 있게 된다.
- copy & paste 를 줄이고 수정 하다가 실수를 줄일 수 있으니 얼마나 대단한 기능 인가! 

### Using roles

```yaml
---
- hosts:
    - all
  vars:
    deploy_root: /home/deploy
    pinpoint_config_root: "{{ deploy_root }}/conf/pinpoint"
  roles:
    - pinpoint-config
```

- a.yml에서는 사용하려는 roles를 넣어주기만 하면 쉽게 설정을 할 수 있다.

### Roles 구조
- roles 하위에 폴더를 여러개 설정할 수 있다.
- tasks, handlers, defaults, vars, files, templates, meta를 설정할 수 있다.
- 대부분 사용하는 것은 tasks, files, templates 정도가 사용될거라 생각한다.

#### tasks
- 하려는 task에 대해서 작성하는 부분이다.
- main.yml은 필수로 존재해야 한다.

#### files
- static files을 모아놓는 폴더이다.

#### templates
- `.jinja2`로 끝나는 파일들을 이용해 `vars`로 동적 파일을 생성 할때 사용한다.

## Jinja2
- ansible은 동적으로 template을 생성하기 위해서 jinja2를 사용한다.

### 문법
- {{ page.jinja2_msg2 }} : 변수를 사용할 때 사용
- `해당 post의 title은 {{ page.jinja2_msg3 }} 이다.`
 
```jinja2
해당 post의 title은 {{ page.title }} 이다.
```

- {{ page.jinja2_msg }} : if, for문 같은 제어문을 사용할 때 사용
- 제어문(if)
    - if, elif, else, endif를 사용해서 제어문을 사용할 수 있다.
    - if, endif는 필수이다.
- 반복문(for ~ in)
    - for item in list, endfor를 사용해서 반복문을 사용할 수 있다.
    
    
## 마치며
- jekyll을 이용하니 jinja2에서 사용하는 문법이랑 같아 표현을 하려니 없는 값으로 표시 된다.
- 예전에 한번 사용했을 때는 엄청 복잡하다고 느껴졌는데, 지금와서 공부해보니 생각보다 쉬운 내용인거 같다.
- 역시 눈으로 볼때랑 직접 해본 다음의 차이가 큰거 같다. 
