---
title: 어댑터 패턴 (Adapter Pattern)
author: keencho
date: 2021-08-21 20:12:00 +0900
categories: [Design Pattern]
tags: [Design Pattern, Java]
---

# **어댑터 패턴**
어댑터 패턴은 클래스의 인터페이스를 사용자가 기대하는 다른 인터페이스로 변환하는 패턴으로, 호환성이 없는 인터페이스 때문에 함께 동작할 수 없는 클래스들이 함께 동작하도록 해주는 패턴입니다.

## **구조**
### 객체 어댑터
![structure](/assets/img/custom/design-pattern/adapter/structure.png)

1. 클라이언트는 비즈니스 로직을 포함하고 있는 클래스입니다.
2. 클라이언트 인터페이스는 다른 클래스가 클라이언트 코드와 협력할 수 있도록 하기 위해 따라야 하는 프로토콜을 설명합니다.
3. 서비스는 유용한 클래스(대체적으로 써드파티 혹은 레거시 코드) 입니다. 클라이언트는 호환되지 않는 인터페이스를 가지고 있기 때문에 이 클래스를 직접 사용할 수 없습니다.
4. 어댑트는 클라이언트와 서비스 모두에서 작동할 수 있는 클래스입니다. 이 클래스는 서비스 객체를 감싸는 동안 클라이언트 인터페이스를 구현합니다.
5. 클라이언트 코드는 클라이언트 인터페이스를 통해 어댑터와 작동하기 때문에 어댑터 클래스와 결합되지 않습니다.

### 클래스 어댑터
![structure2](/assets/img/custom/design-pattern/adapter/structure2.png)
1. 클래스 어댑터는 클라이언트와 서비스의 동작을 직접 상속하므로 객체를 감쌀 필요학 없습니다.

## **적용 가능한 경우**
1. 레거시 코드를 사용하고 싶지만 새로운 인터페이스가 레거시 코드와 호환되지 않을 때 어댑터 클래스를 고려해볼 수 있습니다.
2. 슈퍼클래스에 추가할 수 없는 일부 공통 기능이 없는 기존 서브 클래스를 재사용하려면 이 패턴을 사용해보세요.

## **장단점**
#### **장점**
1. 인터페이스 또는 데이터 변환 코드를 프로그램의 비즈니스 로직과 분리할 수 있기 때문에 단일 책임 원칙을 만족합니다.
2. 기존 클라이언트의 코드를 건들지 않고 클라이언트 인터페이스를 통해 어댑터와 작동하기 때문에 계방/폐쇄 원칙을 만족합니다.

#### **단점**
1. 새로운 인터페이스와 클래스 세트를 구현해야 하기 땜누에 코드의 복잡성이 증가합니다. 나머지 코드와 잘 작동하도록 서비스 클래스를 직접 변경하는 것이 오히려 더 간단할수 있습니다.

## **예제**
모니터와 TV를 컨트롤하는 프로그램을 만들어보겠습니다. 모니터는 on, off의 기능을 가지고있고 TV는 on, off, 볼륨 업, 볼륨 다운의 기능을 가지고 있습니다.

```java
public interface Monitor {
    void on();
    void off();
}

public class LEDMonitor implements Monitor{

    @Override
    public void on() {
        System.out.println("LED Monitor ON");
    }

    @Override
    public void off() {
        System.out.println("LED Monitor OFF");
    }
}

public class LCDMonitor implements Monitor{

    @Override
    public void on() {
        System.out.println("LCD Monitor ON");
    }

    @Override
    public void off() {
        System.out.println("LCD Monitor OFF");
    }
}
```
```java
public interface TV {
    void on();
    void off();
    void volumeUp();
    void volumeDown();
}

public class LEDTV implements TV{

    @Override
    public void on() {
        System.out.println("LED TV ON");
    }

    @Override
    public void off() {
        System.out.println("LED TV OFF");
    }

    @Override
    public void volumeUp() {
        System.out.println("LED TV VOLUME UP");
    }

    @Override
    public void volumeDown() {
        System.out.println("LED TV VOLUME DOWN");
    }
}
```
간단한 코드로 모니터와 TV를 만들었습니다. 근데 모니터로 TV를 볼수 있도록 해달라는 요청사항이 생겼습니다. 어댑터 클래스를 사용하여 모니터에서 TV의 기능을 수행할 수 있도록 해보겠습니다.

```java
public class MonitorAdapter implements TV{

    Monitor monitor;

    public MonitorAdapter(Monitor monitor) {
        this.monitor = monitor;
    }

    @Override
    public void on() {
        monitor.on();
    }

    @Override
    public void off() {
        monitor.off();
    }

    @Override
    public void volumeUp() {
        throw new RuntimeException("지원하지 않는 기능입니다.");
    }

    @Override
    public void volumeDown() {
        throw new RuntimeException("지원하지 않는 기능입니다.");
    }
}
```
모니터로 TV의 on, off 기능을 사용할 수 있게 하였으나, 볼륨 조절 관련 부분은 모니터 하드웨어 상으로 지원하지 않기 때문에 예외를 던짐으로써 해당 메소드의 기능을 제한하였습니다.

```java
public class AdaptorClient {
    public static void main(String[] args) {
        LCDMonitor lcdMonitor = new LCDMonitor();
        LEDMonitor ledMonitor = new LEDMonitor();

        System.out.println("===========================");

        lcdMonitor.on();
        lcdMonitor.off();

        System.out.println("===========================");

        ledMonitor.on();
        ledMonitor.off();

        System.out.println("===========================");

        TV monitorTV = new MonitorAdapter(ledMonitor);
        ledMonitorTV.on();
        ledMonitorTV.off();
        ledMonitorTV.volumeUp();
        ledMonitorTV.volumeDown();

        System.out.println("===========================");
    }
}
```
```
===========================
LCD Monitor ON
LCD Monitor OFF
===========================
LED Monitor ON
LED Monitor OFF
===========================
LED Monitor ON
LED Monitor OFF
Exception in thread "main" java.lang.RuntimeException: 지원하지 않는 기능입니다.
	at sycho.java.pattern.adapter.MonitorAdapter.volumeUp(MonitorAdapter.java:23)
	at sycho.java.pattern.adapter.AdaptorClient.main(AdaptorClient.java:23)
```
클라이언트 코드입니다. 모니터 어댑터 클래스로 TV 기능을 하는 모니터를 만들었고, 볼륨 업 메소드를 호출하자 지원하지 않는 기능이라며 에러를 뱉는 모습까지 확인할 수 있습니다.

### **결론**
어댑터 패턴은 레거시 코드에서 신규 코드로의 변환을 도모할때 도움이 되는 패턴입니다. 다만 어댑터 클래스의 수만큼 코드 복잡성이 증가하게 되는 꼴이므로 역시 올바른 설계가 필요합니다.

### **참조**
> [https://refactoring.guru/design-patterns/adapter](https://refactoring.guru/design-patterns/adapter)
> [https://www.geeksforgeeks.org/adapter-pattern/](https://www.geeksforgeeks.org/adapter-pattern/)

