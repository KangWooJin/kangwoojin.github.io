---
title: "[Plugin] Save Actions를 이용해 팀 내 `Coding convetion` 자동 맞추기" 
categories:
  - Programing
tags:
  - Plugin
toc: true
toc_sticky: true
date: 2020-07-19 08:00:00+09:00 
excerpt: Save Actions를 이용해 Coding convetion을 자동으로 맞춰보자~
---

## 들어가며
개발을 하다보면 code convetion이 맞지 않아 같은 부분을 계속 수정하는 경우가 발생 하는 문제가 있다..

각 팀에서 맞추기로 얘기가 되어 있으면 안 맞춘 사람보고 맞추자고 하면 될텐데 그게 아니라면
지속적인 변경이 발생할 수 밖에 없다..

팀 내에 IDE가 Intellij를 사용한다면 Intellij plugin인 Save Actions를 이용해서 간단하게
code convetion을 적용해 보는 것은 어떨까?

## Install Save Actions

- Intellij에 `Preference`에 들어가게 되면 `Plugins`가 존재한다.
- `Plugins -> Marketplace`에 Save Actions 을 검색하여 다운로드 받는다.

![save-actions](/assets/images/tools/save-acvtions.png)

## Configurations

### General
![save-actions-general](/assets/images/tools/save-action-general.png)

- Save Actions를 동작 관련 부분을 제어한다.
- Active save actions on save의 경우 intellij에서 focus가 다른 곳으로 넘어간 경우
실행 된다.
- Activate save actions on shortcut의 경우는 단축키로 지정된 것을 직접 수행할 때 실행 된다.
- Active save actions on batch는 .. 정확하게 언제 동작하는지 모르겠다..
- No Action if compile errors는 말 그대로 컴파일 에러가 있으면 동작하지 않겠다이다.

> 중요한 부분은!! 팀 내에 어느정도 강제성을 부여하려면, 첫 번째 옵션을 활성화 해두어서
자기도 모르게, code convetion을 지키게 하는 것이다!! 


### Formatting actions
![save-acvtion-formatting](/assets/images/tools/save-acvtion-formatting.png)
- 여기서 부터 intellij에 `code style`이 정의된 xml을 이용해 동작하게 된다.
- Optimize imports의 경우는 사용하지 않는 import를 제거 해준다.
- Reformat file의 경우는 불필요한 공백들을 제거해 준다.
- Rearrange fields and methods는 static과 일반 method가 있으면 static을 상단에 올려주는 기능을 한다.
- 여기서 추천하는 기능은 optimize imports와 Reformat only changed code이다.
- Rearrange를 하게 되면, 많은 부분이 변경될 수 있고, 일부러 하단에 배치해둔 부분을 상단으로 올려 누군가는 
불편해 할 수 있기 때문이다..

### Build actions
![save-acvtion-build](/assets/images/tools/save-acvtion-build.png)
- 아직 실험적인 기능이라.. 확인을 하지 않았다..

### Java inspection and quick fix
![save-acvtion-java-inspection](/assets/images/tools/save-acvtion-java-inspection.png)

- 여기서는 추가적인 기능들을 설정할 수 있는데, 여기 있는 기능들은 필요한 부분만 체크를 해서 사용하면 된다.
- 다른 사람들 코드에 영향을 갈 수 있는 부분이 있기에 다 같이 얘기하고 정의가 되면 좋을 것 같다.
 
### File path inclusions/exclusions
![save-acvtion-file-path](/assets/images/tools/save-acvtion-file-path.png)

- 변경이 필요한 파일들, 변경이 안됬으면 하는 파일들을 지정할 수 있다.
- 자세한 설명은 [Save Actions](https://github.com/dubreuia/intellij-plugin-save-actions#activation)에서 
확인이 가능하다.

## Recommend

![save-action-recommend](/assets/images/tools/save-action-recommend.png)

- 내가 생각하는 추천 방식은 해당 이미지에 체크가 된 부분이다.
- intellij에 내장되어 있는 commit tool을 이용하게 되면, save actions의 기능을 기본적으로
사용할 수 있다.
- 그렇지만 사람들마다 terminal, sourceTree, intellij 등 많은 git tool을 사용하기에
강제할 수는 없으니, intellij에서 포커스가 벗어나는 순간 auto save action이 동작하게 하는 설정이다.

## 마치며
- 많은 사람들이 작업을 해야 하는 코드에 서로 다른 code convetion 설정으로 conflict이 나거나
불필요한 수정이 계속 발생하는 부분을 해결해서 조금 이나마 개발에 더 집중할 수 있는 환경을 만들 수 있었으면 좋겠다.
- 해당 부분을 도입하자고 하면, 분명히 반대하는 사람들이 나올 것인데 해당 팀에서 발생하는 문제점들과
적용되면 좋은 점들을 정리해서 설득을 해보는 것도 하나의 도전이 아닐까 생각한다.


- - -  