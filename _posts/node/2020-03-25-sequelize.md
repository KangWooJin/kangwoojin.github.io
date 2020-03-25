---
title: "[Node.js] query를 편하게 해주는 sequelize에 대해서 알아보자"
categories:
  - Programing
tags:
  - Node
toc: true
toc_sticky: true
date: 2020-03-25 22:00:00+09:00 
excerpt: node 환경에서 사용가능한 query assist 라이브러리인 sequelize에 대해서 알아보자.
---

## 들어가며
slack bot을 만들면서 db query를 사용했어야 했는데, 정말 간단하게 사용가능하며
Java의 JPA처럼 쓸 수 있는 라이브러리인 [sequelize](https://sequelize.org/master/manual/getting-started.html)에 대해서 간단하게 알아보자.

## Getting Started
Node.js를 생성하고 sequelize 라이브러리를 install 해야 한다.

```javascript
npm install --save sequelize
```

그 후에 사용할 데이터 베이스 관련해서 install해야 하며 in-memory 환경에서 간단하게 사용해 볼 예정이라
sqlite3를 install을 한다.

```javascript
$ npm install --save sqlite3
```

## 데이터 베이스 설정
sequelize에서는 다양한 데이터베이스 환경을 지원해주고 있어 필요한 라이브러리를 install해서 해당
설정에 맞게 셋팅을 해주면 된다.

가이드에서는 3가지 방법을 제시해주고 있는데 그 중 편한 방식으로 셋팅하면 된다.

```javascript
const sequelize = new Sequelize('sqlite::memory:'); // Example for sqlite

(async () => {
  try {
    await sequelize.authenticate();
    console.log('Connection has been established successfully.');
  } catch (error) {
    console.error('Unable to connect to the database:', error);
  }
})();
```

해당 방식으로 memory 방식으로 서버가 띄울때 데이터베이스에 연결이 되며 authenticate를 통해서
간단한 테스틑 query를 통해서 연결 유무를 확인할 수 있다.

```sql
SELECT 1+1 AS result
```

## 테이블 생성
Slack bot을 만들며 사용했던 Task table을 어떻게 하면 만들 수 있는지 알아보자.

테이블도 정의할 수 있는 방법이 2가지가 있는데 extending model과 define 방식이 있다.

그 중 extending 방식을 사용해서 해당 테이블용 메소드를 만들어서 사용하는게 더 편해서 해당 방식으로 선택하였다.

```javascript
const {Model, DataTypes, Op} = require('sequelize');

class Task extends Model {
  // method 작성
}

Task.init({
  id: {
    type: DataTypes.STRING(255),
    allowNull: false,
    primaryKey: true
  },
  username: {
    type: DataTypes.STRING(255),
    allowNull: false
  },
  date: {
    type: DataTypes.STRING(255),
    allowNull: false
  },
  task: {
    type: DataTypes.STRING(3000)
  }
}, {/*option*/sequelize, tableName: 'task', timestamps: false});
```

`Model`을 상속 받아서 작성해야 하며, 해당 Class를 init을 통해서 설정해주어야 한다.
nullable, pk 등 다양한 설정을 할 수 있다.

주의해야 할 점은 tableName이 복수형으로 자동 생성 되는 기능이 있는데 `Task`로 클래스를 해두었다면
`Tasks`로 테이블이 생성되니 사용할 때 주의해야 한다.

복수형으로 생성되지 않게 하려면 마지막 Option 부분에 tableName을 재정의해주면 해당 값으로 테이블이 생성 된다.

그리고 sequelize는 테이블을 만들게 되면 `createdAt`, `updatedAt` 필드가 자동으로 추가되는데 해당 필드가 생성되지 않게 하려면
option 부분에 `timestamps: false`을 추가해야 한다.


어느정도 테이블 구조를 잡았으니 실제 데이터베이스에도 동기화를 해주기 위해 sync method를 사용해서 실제로 테이블을 생성해주어야 한다.
```javascript
sequelize.sync(); // 전체 model 동기화
Task.sync(); // 해당 model만 동기화
```

JPA 처럼 create, drop-create등 옵션이 존재하니 잘 사용하면 좋을 것 같다.
```javascript
sequelize.sync({force: true}); // drop-create 방식
sequelize.sync({alter: true}); // 변경된 필드 반영
sequelize.sync(); // 해당 모델의 테이블이 존재하지 않으면 create
```

```sql
Executing (default): CREATE TABLE IF NOT EXISTS `task` (`id` VARCHAR(255) NOT NULL PRIMARY KEY, `username` VARCHAR(255) NOT NULL, `date` VARCHAR(255) NOT NULL, `task` VARCHAR(3000));
```
default는 table이 존재하지 않으면 create 해주는 옵션을 사용한다.

## Insert
Insert는 create를 이용해서 할 수 있으며, json 방식으로 해당 값을 채워주기면 하면 알아서 query가 생성되서 전송된다.

```javascript
(async () => {
  let task = await Task.create({id:'0', username: 'woojin', date:'2020-03-25', task:'sequelize study'});
  console.log('task', task);
})();
```

쿼리 실행

```sql
Executing (default): INSERT INTO `task` (`id`,`username`,`date`,`task`) VALUES ($1,$2,$3,$4);
```

## Select
Select에 대해서도 findAll, findByPk 등 만들어져 있는 메소드들이 존재해 필요한 상황에 따라서 사용하면 된다.

가장 좋았던 거는 `where`절에 대해서 QueryDsl처럼 쉽게 셋팅할 수 있었고 쉽게 어떤 조건인지 확인을 할 수 있었다.

```javascript
(async () => {
  let tasks = await Task.findAll({
    where:  {
      username: 'woojin'
    }
  });
  console.log('tasks', tasks);
})();
```

```sql
Executing (default): SELECT `id`, `username`, `date`, `task` FROM `task` AS `Task` WHERE `Task`.`username` = 'woojin';
```
해당 select문은 username이 'woojin'인 Task를 전체 찾는 쿼리인데 정말 쉽게 원하 조건을 사용할 수 있다.

좀 더 복잡한 쿼리를 만들기 위해서는 해당 [문서](https://sequelize.org/master는/manual/model-querying-basics.html)를 참고하자.

## 마치며
간단하게 토이 프로젝트에 적용해본거라 1:N과 같은 양방향 관계에 대해서는 사용해보질 않았다.

그렇지만 사용하는데 있어서 간단하게 적용해볼 수 있고 다양한 데이터베이스를 지원하고 있어서
데이터베이스에 맞는 쿼리 작성을 하지 않아도 알아서 해당 데이터베이스에 맞게 적용되기에 편하다고 생각한다.

반복적인 쿼리를 만들기 보다는 ORM을 통해서 생산성을 높이는데 집중할 수 있었던 것 같다.

- - - 
[sequelize](https://github.com/KangWooJin/slack-daily-bot) example은 해당 github에서 확인 가능합니다.

 