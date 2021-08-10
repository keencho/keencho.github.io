---
title: 프록시 패턴 (Proxy Pattern)
author: keencho
date: 2021-07-03 20:12:00 +0900
categories: [Design Pattern]
tags: [Design Pattern, Java]
---

# **프록시 패턴**  
프록시(Proxy) 는 `대리자` 라는 뜻입니다. 실생활서의 의미처럼 프로그램에서의 프록시도 `누군가에게 어떤 일을 대신 시키는 것` 이란 의미를 가지고 있습니다.  

때때로 우리는 객체에 대한 액세스를 제어하는 기능을 필요합니다. 예를 들어 무겁고 많은 자원을 필요로 하는 클래스의 한두가지 메소드만 사용해야 하더라도 생성자로 전체 클래스를 인스턴스화 합니다. 
이 시점에 프록시 패턴을 사용할 수 있습니다. `무거운 객체가 상속하고 있는 인터페이스를 상속하는 가벼운 객체를 하나 만드는 방식`입니다. 
이때 가벼운 객체를 `프록시` 라고 하며 우리는 이 가벼운 객체를 인스턴스화하여 사용하고 실제 무거운 행위를 하는 메소드는 정말 그 메소드가 필요한 시점에 호출할 것입니다.  


<br/>
## **적용 가능한 경우**
1. 지연 초기화(가상 프록시) - 가끔 필요하지만 항상 메모리에 적재되어 있는 무거운 서비스 객체가 있는 경우 
  - 서비스가 시작될 때 객체를 생성하는 대신에 객체 초기화가 실제로 필요한 시점에 초기화될수 있도록 지연할 수 있습니다.
2. 액세스 제어(보호 프록시) - 특정 클라이언트만 서비스 객체에 접근 가능하도록 하려는 경우
  - 프록시 객체를 통해 클라이언트의 자격 증명이 기준과 일치하는 경우에만 서비스 객체에 요청을 전달할 수 있습니다.
3. 원격 서비스의 로컬 실행(리모트 프록시) - 서비스 객체가 원격 서버에 위치해 있는 경우
  - 프록시 객체는 네트워크를 통해 클라이언트의 요청을 전달하여 네트워크와 관련된 불필요한 작업들을 처리하고 결과값만 반환합니다.
4. 로깅 요청(로깅 프록시) - 서비스 객체에 대한 기록을 유지하려는 경우
  - 프록시 객체는 서비스 객체에 요청을 전달하기 요청을 기록할 수 있습니다.
5. 캐싱 요청 결과(캐싱 프록시) - 서비스 객체의 결과값이 큰 경우 프록시는 클라이언트 요청의 결과를 캐싱하고 이 캐시의 수명 주기를 관리해야 하는 경우
  - 프록시는 항상 동일한 결과를 생성하는 반복 요청에 대해 캐싱을 구현할 수 있습니다. 또 클라이언트 요청의 매개변수를 캐시 키로 사용할 수 있습니다.
6. 스마트 참조 - 무거운 객체를 사용하는 클라이언트가 없을 때 메모리에서 해제해야 하는 경우
  - 프록시는 서비스 객체의 결과에 대한 참조를 얻은 클라이언트를 추저할 수 있습니다. 때때로 프록시는 클라이언트로 넘어와 아직 클라이언트가 활성 상태인지 확인합니다. 
만약 클라이언트의 리스트가 비어있을 경우 프록시는 서비스 객체를 종료하고 불필요한 시스템 자원을 메모리에서 해제할 수 있습니다.

<br/>
## **장단점**  
#### **장점**
1. 클라이언트가 알지 못하는 상태에서 서비스 객체를 제어할 수 있습니다.
2. 클라이언트가 신경쓰지 않을 때 서비스 객채의 생명 주기를 관리할 수 있습니다.
3. 프록시는 서비스 객체가 준비되지 않았거나 사용할 수 없을때도 작동합니다.
4. OCP(개방-폐쇄 원칙)에 따라 서비스나 클라이언트를 변경하지 않고 새 프록시를 생성할 수 있습니다.

#### **단점**
1. 많은 프록시 클래스를 생성해야 하므로 코드가 더 복잡해질 수 있습니다.
2. 프록시 클래스 자체에 들어가는 자원이 많다면 서비스로부터의 응답이 오히려 늦어질 수 있습니다.

<br/>
## **예제**
이미지 뷰어 프로그램을 만들어보겠습니다. 이미지 뷰어는 다음과 같은 동작을 수행합니다.  

> 이미지 뷰어는 고해상도의 이미지를 불러와 사용자에게 보여줘야 합니다.  
> 사용자가 목록에서 이미지를 선택하기 전까지 실제 이미지를 표시할 필요는 없습니다.

![diagram](/assets/img/custom/design-pattern/proxy/diagram.png)

다음 코드는 이미지 인터페이스 입니다. 이미지를 렌더링하기 위해 구현체가 구현해야 하는 추상메소드 showImage()가 있습니다.
```java
public interface Image {
    void showImage();
}
```

다음 코드는 우리가 현재 사용하고 있는 이미지 인터페이스의 구현체입니다. 디스크에서 고해상도 이미지를 불러와 메모리에 적재하고 showImage()가 호출되면 화면에 렌더링 합니다.
```java
public class HighResolutionImage implements Image {

    public HighResolutionImage(String path) {
        loadImage(path);
    }

    private void loadImage(String path) {
        // 이미지를 디스크에서 불러와 메모리에 적재
        // 작업 자체가 무겁고 많은 자원을 필요로함
    }

    @Override
    public void showImage() {
        // 이미지를 화면에 렌더링
    }
}
```

다음 코드는 이미지 뷰어 메인 클래스입니다.  
```java
public class ImageViewer {
    public static void main(String[] args) {
        Image highResolutionImage1 = new HighResolutionImage("highResolutionImage1");
        Image highResolutionImage2 = new HighResolutionImage("highResolutionImage2");
        Image highResolutionImage3 = new HighResolutionImage("highResolutionImage3");

        highResolutionImage2.showImage();
    }
}
```
사용자가 3개의 이미지가 있는 폴더를 선택하였습니다. 사용자가 어떤 이미지를 선택할지 몰라 모든 이미지를 인스턴스화 하였습니다. 실제로는 highResolutionImage2 만 사용자에게 보여줘야 함에도 불구하고 `HighResolutionImage` 클래스의 생성자에 의해 모든 이미지는 `loadImage()` 메소드를 호출하게 됩니다. 
이 시점에서 불필요한 자원낭비가 발생하였습니다.  

자, 이제 이미지 인터페이스를 구현하는 가벼운 프록시 클래스를 만들어보겠습니다. 
```java
public class ImageProxy implements Image {

    private String path;
    private Image proxyImage;

    public ImageProxy(String path) {
        this.path = path;
    }

    @Override
    public void showImage() {
        proxyImage = new HighResolutionImage(path);

        proxyImage.showImage();
    }
}
```
이 클래스는 `showImage()` 메소드가 호출되는 시점에 `HighResolutionImage` 클래스를 인스턴스화 합니다.  

이미지 뷰어 메인 클래스의 코드를 변경하겠습니다.
```java
public class ImageViewer {
    public static void main(String[] args) {
        Image proxyHighResolutionImage1 = new ImageProxy("highResolutionImage1");
        Image proxyHighResolutionImage2 = new ImageProxy("highResolutionImage2");
        Image proxyHighResolutionImage3 = new ImageProxy("highResolutionImage3");

        proxyHighResolutionImage1.showImage();
    }
}
```
객체의 타입은 `Image` 로 동일하지만 `HighResolutionImage` 클래스 대신 `ImageProxy` 객체로 인스턴스화 하였습니다.  

첫번째 메인 클래스의 경우와는 다르게 객체를 인스턴스화 하는 시점에 `HighResolutionImage` 클래스의 `loadImage()` 메소드를 호출하지 않습니다. 
`proxyHighResolutionImage1.showImage()`를 호출하는 시점에 `loadImage()` 메소드를 호출하게 됩니다. 실제 메소드를 호출하는 시점에 메모리 적재가 이루어지기 때문에 불필요한 자원낭비가 발생하지 않게 됩니다.

<br/>
### **결론**
프록시 패턴은 많은 곳에서 쓰이고 있습니다. 핵심은 `클라이언트 -> 프록시 -> 실제 객체` 로 구성된 구조라는 점입니다. 매우 간단한 패턴이지만 자칫 잘못하면 불필요한 코드로 오히려 복잡해 질수 있으니 적재적소에 활용하는 편일 좋을것 같습니다.  

<br/>
### **참조**
> [https://refactoring.guru/design-patterns/proxy](https://refactoring.guru/design-patterns/proxy)  
> [https://www.oodesign.com/proxy-pattern.html](https://www.oodesign.com/proxy-pattern.html)
