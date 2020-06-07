---
title: "[Vuejs] Vue.js tutorial(1) - project 셋팅" 
categories:
  - Programing
tags:
  - Vuejs
toc: true
toc_sticky: true
date: 2020-06-07 11:00:00+09:00 
excerpt: 누구나 쉽게 따라할 수 있는 Vue.js tutorial을 통해서 오픈소스에 기여 해보자.
---

## 들어가며
vue.js에 대해서 프로젝트 셋팅에서 부터 오픈소스에 기여하기까지를 블로그에 기록을 할 예정이다.

시작은 vue.js에 대한 설명과 프로젝트 셋팅에 대해서 알아보자~

## Node 설치하기
- vue cli를 사용하기 위해서는 Node.js가 설치가 되어 있어야 한다.
- vue cli 4를 이용하기 위해선 Node.js의 최소 버전은 8.9 이상이어야 한다.
- 각 OS 환경에 맞게 LTS로 설치 한다. 
- [Go to Download Node](https://nodejs.org/ko/download/)

## vue-cli 설치하기

- vue-cli를 통해서 프로젝트를 간단하게 셋팅할 수 있다. 
- vue-cli는 현재 포스트 작성 기준 4.4.1까지 나와 있으며 vue-cli 4 기준으로 작성 한다.
- global로 vue cli를 설치하다 보니 sudo 권한을 통해서 권한을 물어볼 수 있으니 참고!
- terminal에서 해당 명령어를 입력하여 vue cli를 global로 설치한다.

```
npm install -g @vue/cli
# OR
yarn global add @vue/cli
```

- 그 후 설치가 정상적으로 되었는지 확인을 위해서 version check를 진행한다.
- version이 4.x.x 이상이면 따라하는데 문제가 없을 것이다~!

```
vue --version
```

## vue project 셋팅하기
- vue가 설치가 완료되었다면 vue project를 생성해야 한다.
- vue project는 아래 command로 생성 할 수 있다.

```
vue create vue-example
```

- 해당 command를 입력하면 아래와 같은 이미지가 나온다.

![vue-create-default](/assets/images/vuejs/vue-create-default.png)

- default는 babel, eslint만 적용되고 나머지는 각자가 추가해야 한다.
- Manually select features를 통해서 상세 설정을 추가할 수 있다.
- 상세 설정을 추가 한 뒤에 설정 값을 저장해서 다른 프로젝트를 생성할 때 재활용 할 수 있다.

![vue-manually-select-features](/assets/images/vuejs/vue-manually-select-features.png)

- Manually select features를 선택하게 되면 아래와 같이 설정하고 싶은 값들이 나온다.
- 하나씩 공부를 하면되니 전체 옵션을 선택하고 적용해본다.

### Manually select features

#### Use class-style component syntax?

- 클래스 스타일 Vue 컴포넌트를 사용할지인데, 추천값이 Y이므로 추가한다.
- [vue-class-compoent](https://kr.vuejs.org/v2/guide/typescript.html#%ED%81%B4%EB%9E%98%EC%8A%A4-%EC%8A%A4%ED%83%80%EC%9D%BC-Vue-%EC%BB%B4%ED%8F%AC%EB%84%8C%ED%8A%B8) 데코레이터를 사용하는 방식인데, 현재는 공식적인 라이브러리가 되었는 것 같다.
- [Vue Property Decorator](https://github.com/kaorun343/vue-property-decorator) 에 대해서는 github에서 설명이 추가 되어 있다.

#### Use Babel alongside TypeScript (required for modern mode, auto-detected polyfills, transpiling JSX)?
- TypeScript에 babel 설정을 할 것인지 물어보는 설정이고 추천 값이 Y이므로 추가한다.

#### Use history mode for router? (Requires proper server setup for index fallback in production) (Y/n) 
- [router의 설정 값](https://router.vuejs.org/kr/guide/essentials/history-mode.html#html5-%ED%9E%88%EC%8A%A4%ED%86%A0%EB%A6%AC-%EB%AA%A8%EB%93%9C) 인데 history mode와 hash mode가 존재한다.
- history mode : URL이 변경될 때 페이지를 다시 로드하기에 URL에 대해서 `index.html`로 전달해주는 서버 설정이 필요하다. 
- hash mode : URL 해시를 사용하여 전체 URL을 시뮬레이트하므로 URL이 변경될 때 페이지가 다시 로드 되지 않는다.

#### Pick a CSS pre-processor (PostCSS, Autoprefixer and CSS Modules are supported by default): (Use arrow keys)
- Css 전처리기를 설정하는 부분인데, `Node-sass`로 선택한다.
- 다른 옵션들도 많이 있지만 size가 가장 작은 node-sass로 선택했다.
- [trends 분석](https://www.npmtrends.com/node-sass-vs-less-vs-dart-sass-vs-stylus)

#### Pick a linter / formatter config
- eslint와 formatter를 설정하는데, tslint는 deprecated되어 lint는 eslint만 된다.
- formatter는 [trends 분석](https://www.npmtrends.com/airbnb-style-vs-standard-vs-prettier-eslint) 을 하니 prettier가 가장 다운로드 수가 높아서 선택하였다.

#### Pick additional lint features
- Lint on save는 저장시 lint 검사를 제공한다.
- Lint and fix on commit는 저장시 자동으로 fix가 가능한 것은 Fix까지 해준다.
- 따라서 두 가지 옵션을 다 추가하였다.

#### Pick a unit testing solution
- unit test인데 test를 한번도 해보질 못해서 잘 모르기에 `Mocha + Chai`로 선택한다.

#### Pick an E2E testing solution
- E2E testing으로 추후 공부를 위해서 추가한다.
- `Cypress`로 선택!

#### Where do you prefer placing config for Babel, ESLint, etc.?
- 설정 값들을 config 파일로 관리할지, package.js 에서 관리할지 고르는건데, package.js로 관리하는게 편해보인다.

## 시작하기

- 기본적인 설정을 다 했기에 vue를 시작해서 정상동작하는지 확인을 할 차례이다.
- 아래 명령어를 통해서 정상동작하는지 확인 한다.
```
npm run serve
```

- [http://localhost:8080](http://localhost:8080) 에 접근해 보자.
- 처음 시작하게 되면 welcome 메시지를 가진 페이지가 나타난다.

![vue-welcome](/assets/images/vuejs/vue-welcome.png)


## 마치며
vue js에 대해서 project 셋팅을 하면서 간단하게 설정 값들에 대해서 알아보았다.

이제 개발을 진행할 수 있는 상태가 되었으니, 필요한 문법과 라이브러리에 대해서 하나씩 공부해보자.

- - - 
[Vue.js](https://github.com/KangWooJin/spring-study/tree/master/vuejs)
관련 example code는 github에 올려두었으니 참고~!