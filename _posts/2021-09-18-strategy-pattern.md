---
title: 전략 패턴 (Strategy Pattern)
author: keencho
date: 2021-09-18 20:12:00 +0900
categories: [Design Pattern]
tags: [Design Pattern, Java]
---

# **전략 패턴**
전략 패턴은 실행 중에 알고리즘을 선택할 수 있게 하는 행위 디자인 패턴 입니다. 즉, 각각의 객체들이 할 수 있는 동작들일 미리 전략으로 만들어 놓고 동작을 동적으로 변경해야 한다면 해당 전략만 변경하여 객체의 동작이 바뀌도록 하는 패턴입니다.

<br/>
## **구조**
![structure](/assets/img/custom/design-pattern/strategy/structure.png)

1. 컨텍스트는 전략 구현체중 하나와 참조를 유지하고 전략 인터페이스를 통해서만 객체와 통신합니다.
2. 전략 인터페이스는 모든 전략 구현체에 대한 공용 인터페이스입니다. 컨텍스트가 전략을 실행하는데 사용하는 메소드를 선언합니다.
3. 인터페이스의 구현체는 컨텍스트가 사용하는 알고리즘의 다른 버전을 구현합니다.
4. 컨텍스트는 알고리즘을 실행해야 할 때마다 해당 알고리즘과 연결된 전략 객체의 메소드를 호출합니다. 컨텍스트는 어떤 전략이 동작하는지, 혹은 어떤 알고리즘이 실행되는지 알 필요는 없습니다.
5. 클라이언트는 특정 전략 객체를 만들어 컨텍스트에 전달합니다.

<br/>
## **적용 가능한 경우**
1. 객체 내에서 서로 다른 알고리즘을 사용하고 런타임시 다른 알고리즘으로 전환하고자 하는 경우
2. 일부 행위를 동작하는 방법만 다른 비슷한 클래스가 많은 경우
3. 비즈니스 로직을 알고리즘의 세부 내용에서 분리하고자 하는 경우

<br/>
## **장단점**
#### **장점**
1. 런타임에 객체 내부에서 사용되는 알고리즘을 바꿀수 있습니다.
2. 알고리즘을 사용하는 코드에서 이를 구현하는 세부 정보를 분리할 수 있습니다.
3. 상속을 구성으로 대체할 수 있습니다.
4. 기존 컨텍스트를 변경하지 않고 새로운 전략을 도입함으로써 개방/폐쇄 원칙을 만족합니다.

#### **단점**
1. 알고리즘이 많지 않고 자주 변경되지 않는다면 새로운 클래스와 인터페이스를 만들어 프로그램을 복잡하게 만들 이유가 없습니다.
2. 개발자는 적절한 전략을 선택하기 위해 전략 간의 차이점을 알고 있어야 합니다.
3. 최신 프로그래밍 언어에는 클래스 및 인터페이스를 추가하지 않아도 전략 객체를 사용하는 것과 동일한 효과를 낼 수 있는 방법이 있습니다.

<br/>
## **예제**
결제를 할수 있는 환경이 PC / Mobile 로 나뉘어져 있고 각각의 환경은 하나의 결제 방식을 선택할수 있으며 이를 바꿀수 있는 예제를 만들어 보겠습니다.

아래 코드는 결제 전략과 이를 구현한 세가지의 결제 방식(페이) 클래스 입니다.
```java
public interface PayStrategy {
    void pay(int amount);
}

public class NaverPay implements PayStrategy{
    @Override
    public void pay(int amount) {
        System.out.println("네이버 페이로 결제 - 금액 : " + amount);
    }
}

public class KakaoPay implements PayStrategy{
    @Override
    public void pay(int amount) {
        System.out.println("카카오 페이로 결제 - 금액 : " + amount);
    }
}

public class TossPay implements PayStrategy{
    @Override
    public void pay(int amount) {
        System.out.println("토스 페이로 결제 - 금액 : " + amount);
    }
}
```

아래 코드는 결제 전략을 변수로 갖고 setter() 메소드를 통해 결제 전략을 주입할 수 있는 클래스 입니다. 해당 클래스를 통해 결제 환경을 만들 수 있습니다.
```java
public class PayEnvironment {
    private PayStrategy payStrategy;

    public void setPayStrategy(PayStrategy payStrategy) {
        this.payStrategy = payStrategy;
    }

    public void pay(int amount) {
        if (payStrategy != null) {
            payStrategy.pay(amount);
        }
    }
}

public class PC extends PayEnvironment {
}

public class Mobile extends PayEnvironment {
}
```

마지막 클라이언트 코드 입니다. 결제 환경에 따라 결제 전략을 선택할 수 있고 특정 조건을 만족하면 결제 전략을 변경합니다.
```java
public static void main(String[] args) {
    PayEnvironment pc = new PC();
    PayEnvironment mobile = new Mobile();

    pc.setPayStrategy(new KakaoPay());
    mobile.setPayStrategy(new NaverPay());

    pc.pay(10);
    mobile.pay(30);

    if (condition()) {
        mobile.setPayStrategy(new TossPay());
    }

    mobile.pay(50);
}

public static boolean condition() {
    // 조건ㅇ
    return true;
}
```

<br/>
### **결론**
전략 패턴은 현업에서 알게 모르게 많이 사용하고 있는것 같습니다. 요즘 새로운 언어에서는 기본 기능으로도 간단하게 구현할 수 있으니 한번 찾아 보시길 추천 드리겠습니다.

<br/>
### **참조**
> [https://refactoring.guru/design-patterns/strategy](https://refactoring.guru/design-patterns/strategy)

