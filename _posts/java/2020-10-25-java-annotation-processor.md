---
title: "[Java] Annotation Processor 실습" 
categories:
  - Programing
tags:
  - Java
toc: true
toc_sticky: true
date: 2020-10-25 19:00:00+09:00 
excerpt: Annotation Processor를 직접 만들어서 class 파일이 생성되는지 확인해보자.
---

## 들어가며
Annotation Processor를 이용해서 컴파일 시점에 class 파일에 원하는 코드가 생성 될 수 있도록 테스트를 진행 해보자.

## 의존성 주입
```xml
<dependencies>
  <!-- https://mvnrepository.com/artifact/com.google.auto.service/auto-service -->
  <dependency>
    <groupId>com.google.auto.service</groupId>
    <artifactId>auto-service</artifactId>
    <version>1.0-rc7</version>
  </dependency>
  <!-- https://mvnrepository.com/artifact/com.squareup/javapoet -->
  <dependency>
    <groupId>com.squareup</groupId>
    <artifactId>javapoet</artifactId>
    <version>1.13.0</version>
  </dependency>
</dependencies>
```
- `java.lang.Process`에 Processor를 컴파일 시점에 등록을 도와주는 `AutoService`를 추가한다.
- `javapoet`은 클래스를 조작하기 쉽게 Builder로 라이브러리를 제공해 준다.

## Processor 셋팅
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.SOURCE)
public @interface Magic {
}
```

```java
@AutoService(Process.class)
public class MagicMojaProcessor extends AbstractProcessor {

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return Set.of(Magic.class.getName());
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.RELEASE_11;
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(Magic.class);
        for (Element element : elements) {
            Name elementSimpleName = element.getSimpleName();
            if (element.getKind() != ElementKind.INTERFACE) {
                processingEnv.getMessager().printMessage(Kind.ERROR, "Magic annotation can not be used on" + elementSimpleName);
            } else {
                processingEnv.getMessager().printMessage(Kind.NOTE, "processing " + elementSimpleName);
            }


            TypeElement typeElement = (TypeElement) element;
            ClassName className = ClassName.get(typeElement);

            MethodSpec pullOut = MethodSpec.methodBuilder("pullOut")
                                           .addModifiers(Modifier.PUBLIC)
                                           .returns(String.class)
                                           .addStatement("return $S", "Rabbit!")
                                           .build();

            TypeSpec magicMoja = TypeSpec.classBuilder("MagicMoja")
                                         .addModifiers(Modifier.PUBLIC)
                                         .addSuperinterface(className)
                                         .addMethod(pullOut)
                                         .build();

            Filer filer = processingEnv.getFiler();

            try {
                JavaFile.builder(className.packageName(), magicMoja)
                        .build()
                        .writeTo(filer);
            } catch (IOException e) {
                processingEnv.getMessager().printMessage(Kind.ERROR, "FATAL ERROR:" + e);
            }

        }

        return true;
    }
}
```

- `@Magic` annotation이 붙어 있는 interface가 존재한다면 `MagicMoja` class를 생성하는 processor를 만들어보았다.

## class 생성 확인
```xml
<dependencies>
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.11</version>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>kangwoojin.github.io</groupId>
    <artifactId>an-pro</artifactId>
    <version>1.0-SNAPSHOT</version>
  </dependency>
</dependencies>
```

- 다른 Maven 프로젝트를 만들고 해당 라이브러리를 사용할 수 있게 dependency에 추가하였다.

```java
@Magic
public interface Moja {
  String pullOut();
}
```

- Moja라는 interface를 생성하고 @Magic annoation을 추가하고 컴파일을 하게 되면 `MagicMoja` class가 classes에 생성이 된다.
 

## 마치며
- 똑같이 따라 해보았지만, class 파일이 생성은 되지 못 했다.. 이유는 찾아서 다시 작성해야 겠다. 
- `Gradle`에서 처음 따라해보았지만, `AutoService`가 정상 동작하지 않아서 `Maven`에서 테스트를 진행하니 정상적으로 동작하였다.
- `Gradle`에서는 추가 옵션을 줘야 동작 하는 것 같다.
