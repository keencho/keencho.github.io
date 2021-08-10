---
title: 싱글턴 패턴 (Singleton Pattern)
author: keencho
date: 2021-07-06 20:12:00 +0900
categories: [Design Pattern]
tags: [Design Pattern, Java]
---

# **싱글턴 패턴**  
싱글턴 패턴은 생성자가 여러 차례 호출되더라도 실제로 생성되는 객체는 하나이고 최초 생성 이후 호출된 생성자는 
최초의 생성자가 생성한 객체를 리턴하는 디자인 유형입니다.  

<br/>
## **구조**  
![structure](/assets/img/custom/design-pattern/singleton/structure.png)  

1. 싱글턴 클래스는 자체 클래스의 동일한 인스턴스를 반환하는 `getInstance()` 라는 정적 메소드를 선언했습니다.  
  - 싱글턴의 생성자는 클라이언트 코드에서는 확인할 수 없도록 숨겨야 합니다. `getInstance()` 메소드를 호출하는 것 만이 싱글턴 객체를 가져올 수 있는 유일한 방식이어야 합니다.

<br/>
## **적용 가능한 경우**  
1. 프로그램 클래스에 모든 클라이언트가 사용할 수 있는 한개의 인스턴스만 존재해야 하는 경우 싱글턴 패턴을 사용합니다. 예를들어 위에 언급했듯이 DBCP가 있습니다.  
  - 싱글턴 패턴은 특별한 방법을 제외하고 클래스의 객체를 생성하는 다른 모든 수단을 차단합니다. 이 메소드는 이미 생성된 객체가 있는 경우 기존 객체를 반환하고 그렇지 않은 경우에만 새 겍체를 생성합니다.  
2. 전역 변수를 더 엄격하게 컨트롤해야 하는 경우 싱글턴 패턴을 사용합니다.  
  - 전역 변수와 달리 싱글턴 패턴은 클래스의 인스턴스가 하나만 있음을 보장합니다. 싱글턴 클래스 자체를 제외하고 이미 캐시된 인스턴스를 대체할 수 있는 것은 없습니다.  

<br/>
## **장단점**
#### **장점**  
1. 클래스에 하나의 인스턴스만 존재함을 확실시 할 수 있습니다.
2. 해당 인스턴스에 대한 전역 엑세스 포인트를 얻습니다.
3. 싱글턴 객체는 첫 요청시에만 초기화 됩니다.  

#### **단점**  
1. 전역 엑세스 포인트를 얻기 때문에 단일 책임 원칙 (Single Responsibility Principle)을 위반합니다.  
2. 너무 많은 곳에서 사용할 경우 잘못된 디자인 형태가 될 수도 있습니다.  
3. 멀티 스레드 환경에서 특별한 처리가 필요하므로 멀티 스레드 환경에서 싱글턴 객체를 여러번 생성하지 않습니다.  
4. 많은 테스트 프레임워크가 Mock 객체를 생성할 때 상속에 의존하기 때문에 싱글턴의 클라이언트 코드를 테스트하기 어렵습니다.  

<br/>
## **예제**
간단한 싱글턴을 구현해보겠습니다. 단지 할 일은 생성자를 숨기고 정적으로 생성할 메소드를 구현하면 됩니다.
```java
Singleton.java: 싱글턴 객체

public class Singleton {
    private static Singleton instance;
    public String value;

    private Singleton(String value) {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException ex) {
            ex.printStackTrace();
        }
        this.value = value;
    }

    public static Singleton getInstance(String value) {
        if (instance == null) {
            instance = new Singleton(value);
        }
        return instance;
    }
}
```

```java
SingleThreadSingleton.java: 클라이언트 코드 

public class SingleThread {
    public static void main(String[] args) {
        System.out.println("같은 값을 보게 된다면 싱글턴 객체가 재사용된 것입니다..!!" + "\n" +
                "다른 값을 보게 된다면 두개의 싱글턴 객체가 생성된 것입니다..!!");

        Singleton singleton = Singleton.getInstance("x");
        Singleton anotherSingleton = Singleton.getInstance("y");

        System.out.println(singleton.value);
        System.out.println(anotherSingleton.value);
    }
}
```  

위 코드를 실행해보면 다음과 같은 결과를 얻을 수 있습니다.
```
같은 값을 보게 된다면 싱글턴 객체가 재사용된 것입니다..!!
다른 값을 보게 된다면 두개의 싱글턴 객체가 생성된 것입니다..!!
x
x
```  
결과값이 같은 것으로 보아 싱글턴 객체가 재사용된 것을 확인할 수 있습니다.  

위 첫번째 예제는 싱글 스레드 환경에서 싱글턴 패턴을 사용한 경우입니다. 만약 멀티 스레드 환경에서 싱글턴 패턴을 구현하고자 한다면, 두개 이상의 스레드가 인스턴스를 획득하기 위해 `getInstance()` 메소드에 진입하여 경쟁하는 과정에 두개의 인스턴스가 생성되는일이 발생할 수 있습니다. 
그렇다면 멀티 스레드 환경에서 싱글턴 패턴을 사용하고자 할 때 어떤식으로 구현해야 할까요? 예제를 통해 알아보겠습니다.  

첫번째 방법은 DCL(Double Checked Locking) 입니다. 사실 그다지 권고되지 않는 방법입니다.
```java
public static Singleton getInstance(String value) {
    if (instance == null) {
        synchronized(Singleton.class) {
            Singleton inst = instance;
            if (inst == null) {
                synchronized(Singleton.class) {
                    instance = new Singleton(value);
                }
            }
        }
    }
    return instance;
}
```
인스턴스가 null 일 경우 동기화 블럭에 진입해 인스턴스를 생성합니다. 언뜻보기 문제없어 보이는 코드이지만 다음과 같은 문제가 발생할 수 있습니다.  

> 1. 스레드 a, 스레드 b 가 존재한다.
> 2. 스레드 a가 인스턴스를 생성하기 전에 메모리 공간에 할당이 가능하기 때문에 스레드 b가 인스턴스를 생성하기 위해 진입한다. 
> 3. 스레드 a는 스레드 b가 할당된 것을 보고 인스턴스를 사용하려고 하나 스레드 b의 인스턴스 생성과정이 끝난 상태가 아닐수 있기 떄문에 에러가 발생한다.  

위와 같은 문제가 있어 동기화 블록의 사용을 추천하지는 않고 `LazyHolder`기법 사용을 추천합니다.  
```java
public class Singleton {

    private static class LazyHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    private Singleton() { }

    public static Singleton getInstance() {
        return LazyHolder.INSTANCE;
    }
}
```  
위 예제는 `getInstance()` 가 호출될때만 싱글턴 클래스를 초기화하고 100% Thread-safe 합니다. 심지어 클래식한 방법이어서 자바 버전에 구애받지 않고 성능도 꽤 높습니다.  

위 코드가 동작하는 이유는 클래스 로더가 클래스의 정적 초기화를 처리하기 위한 자체 동기화 코드를 갖고 있기 때문입니다. 클래스가 사용되기 전에 정적 초기화가 완료되었음을 보장받을 수 있으며 위 코드에서 클래스는 `getInstance()` 메소드에서만 사용됩니다.
이는 `getInstance()` 메소드를 호출할 때 클래스가 내부 클래스를 로드함을 의미합니다.  

<br/>
### **결론**
싱글턴 패턴은 인스턴스가 한개만 생성되어야 하는 경우에 적합한 패턴입니다. 다만 클라이언트 코드에서 전역 엑세스 포인트를 얻어 너무 많은 곳에서 사용하면 클래스간의 결합도가 높아져 오히려 패턴을 사용 안하느니만 못하게 될 수도 있으므로 주의가 필요합니다.  

<br/>
### **참조**
> [https://refactoring.guru/design-patterns/singleton](https://refactoring.guru/design-patterns/singleton)  
> [https://stackoverflow.com/a/11072311/13160032](https://stackoverflow.com/a/11072311/13160032)
