---
title: "[Nginx] Nginx location directive" 
categories:
  - Programing
tags:
  - Nginx
toc: true
toc_sticky: true
date: 2020-09-13 12:00:00+09:00 
excerpt: Nginx location을 설정할 때 사용되는 설정 방법에 대해서 알아보자.
---

## 들어가며
최근에 nginx을 이용해 다른 front쪽 서버를 proxy해주는 작업을 진행 했었다.

진행을 하면서, api 대해서는 문제없이 proxy가 되었지만, css, js 파일과 같은 static file에 대해서는 proxy가 되지 않고
block 되는 현상이 발생했다.

block이라고 하기보다는 front쪽 서버에 static file을 찾아야 하지만 proxy 해주는 서버에서 static한 파일을 찾는 문제가 발생 했다.

## 배경 설명

```
Client <-> Proxy Server <-> Front Server
```

- 이런식으로 서버가 구축이 되어 있다고 가정한다.
- `http://example.com/front` 요청이 오는 경우 front page를 보여줘야 한다.
- Proxy Server 에는 nginx를 이용해 Front Server로 요청을 전달해 주고 있고, Front Server에 직접 접근을 하였을 때
페이지가 정상 동작하였다.
- 그렇지만 Proxy Server만 거치게 되면 static file을 못 찾는 문제가 발생했다.

## 원인
```
upstream front {
  http://127.0.0.1
}

location /front/ {
  proxy_pass http://front;
  ...
}

location / {
  try_files $uri /index.html;
}
```

- Proxy Server에 있는 nginx 설정의 일부이다.
- 여기까지만 보고도 바로 원인을 알고 있다면 nginx에 대해서 잘 알고 있으니 끝까지 안봐도 될거 같다.
- location 에 regex expression을 사용하지 않고 none 타입으로 사용하였다.
- 이렇게 되는 경우 `http://example.com/front/xxx.js` 요청이 들어오면, `/front`에 매칭이 되지 않고,
`/`에 매칭이 되어 proxy server에서 js 파일을 찾게 된다.

## 해결
```
upstream front {
  http://127.0.0.1
}

location ^~ /front/ {
  proxy_pass http://front;
  ...
}

location / {
  try_files $uri /index.html;
}
```

- `^~` 을 추가하여, regex expression으로 동작할 수 있게 변경하였다.
- `/front/`, `/front/xxx.js`에 일치 일치하는 경우 proxy 되어 front 서버로 요청이 전달되게 된다.

## nginx 간단 정리
### all
```
location / {
    ...
}
```
- 모든 요청에 대해서 매칭이 될 수 있으나, 우선순위가 가장 낮게 되어 있어서 앞에 매칭이 되지 않으면
해당 location 매칭이 사용된다.

### =
```
location = /images { 
    ...
}
```
- 해당 매칭과 정확하게 일치하는 경우만 매칭이 발생한다.
- `http://example.com/images` 는 매칭이 되고, `http://example.com/images/index.html`, `http://example.com/images/`
의 경우는 매칭이 되지 않는다.
  
### ~
```
location ~ /images { 
    ...
}
```
- 대소문자를 구분하여 매칭을 진행 한다.

### ~*
```
location ~* /images { 
    ...
}
```
- 대소문자 구분을 하지 않고 매칭을 진행 한다.

### ^~
```
location ^~ /images { 
    ...
}
```
- 대소문자를 구분하며, `/images`로 시작하는 URI에 대해서 매칭을 진행하고, 최적의 값을 찾고 나면
이후 매칭을 진행하지 않는다.

## 마치며
- 정말 수정한 부분은 별로 안되지만, 동작하는 방식이 많이 달라지기에 사소한 차이를 잘 알고 사용하는게 중요 한 것 같다.
