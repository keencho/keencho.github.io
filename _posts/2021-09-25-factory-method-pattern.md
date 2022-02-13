---
title: 팩토리 메소드 패턴 (Factory Method Pattern)
author: keencho
date: 2021-09-25 20:12:00 +0900
categories: [Design Pattern]
tags: [Design Pattern, Java]
---

# **팩토리 메소드 패턴**
팩토리 메소드 패턴은 부모 클래스에 알려지지 않은 구체 클래스를 생성하는 패턴이며, 자식 클래스가 어떤 객체를 생성할지 결정하도록 하는 생성패턴 입니다. 간단히 요약하자면 부모 클래스 대신 자식 클래스에서 객체를 생성하는 패턴이라고 생각하시면 됩니다.

## **구조**
![structure](/assets/img/custom/design-pattern/factory-method/structure.png)

1. 생성자와 그 하위클래스 객체를 생성하는데 공통적으로 사용될수 있는 `Product` 인터페이스를 선언합니다.
2. `Concreate Products`는 `Product` 인터페이스의 구현체 입니다.
3. `Creator` 클래스는 새로운 product 객체를 반환하는 팩토리 메소드를 선언합니다. 중요한 점은 이 메소드의 리턴 타입이 product 인터페이스와 일치해야 한다는 것입니다.
4. `Concreate Creators`는 다른 타입의 product를 반환할 수 있도록 기본 팩토리 메소드를 재정의 합니다.

## **적용 가능한 경우**
1. 코드가 동작해야 하는 객체의 유형과 종속성을 미리 알수 없는 경우
2. 라이브러리 혹은 프레임워크 사용자에게 구성 요소를 확장하는 방법을 제공하려는 경우
3. 기존 객체를 재구성하는 대신 기존 객체를 재사용하여 리소스를 절약하고자 하는 경우

## **장단점**
#### **장점**
1. 생성자와 구현 클래스의 결합을 피할 수 있습니다.
2. 객체 생성 코드를 한 곳 (패키지, 클래스 등)으로 이동하여 코드를 유지보수하기 쉽게 할수 있으므로 단일 책임 원칙을 만족합니다.
3. 기존 클라이언트 코드를 건드리지 않고 새로운 타입의 객체를 생성할수 있기 때문에 개방/폐쇄 원칙을 만족합니다.

#### **단점**
1. 패턴을 구현하기 위해 많은 서브 클래스를 만들어 구현해야 하기 때문에 코드의 복잡성이 증가할 수 있습니다.

## **예제**
모형의 타입에 따라 어떤 모형을 생성할지 팩토리 클래스에서 정할수 있게 되는 간단한 예제를 만들어 보겠습니다.

일단 모양 인터페이스와 그 구현체들 입니다.
```java
public interface Shape {
    void draw();
}

public class Square implements Shape{
    @Override
    public void draw() {
        System.out.println("draw square");
    }
}

public class Rectangle implements Shape{
    @Override
    public void draw() {
        System.out.println("draw rectangle");
    }
}
```

다음은 팩토리 메소드 패턴의 꽃 팩토리 클래스 입니다. getShape() 메소드는 모형의 타입에 따라 square 혹은 rectangle 인스턴스를 반환합니다.
```java
public class ShapeFactory {
    public Shape getShape(String type) {
        if ("SQUARE".equalsIgnoreCase(type)) {
            return new Square();
        }

        if ("RECTANGLE".equalsIgnoreCase(type)) {
            return new Rectangle();
        }

        return null;
    }
}
```

마지막으로 클라이언트 입니다. 팩토리 클래스에 의해 조건에 맞는 모형 인스턴스를 생성하고 draw() 메소드를 호출합니다.
```java
public class FactoryMethodClient {
    public static void main(String[] args) {
        ShapeFactory shapeFactory = new ShapeFactory();

        Shape s1 = shapeFactory.getShape("SQUARE");
        s1.draw();

        Shape s2 = shapeFactory.getShape("RECTANGLE");
        s2.draw();
    }
}
```

새로운 모형이 생긴다 하더라도 shape 인터페이스를 상속하는 새로운 클래스를 만들고 팩토리 클래스의 getShape() 메소드만 수정해주면 되기 때문에 단일 책임 원칙, 개방/폐쇄 원칙을 만족하게 됩니다.

### **결론**
팩토리 메소드 패턴을 사용하는 궁극적인 이유는 클래스간의 결합도를 느슨하게 가져가게 하기 위함 입니다. 팩토리 클래스를 통해 객체 생성 코드를 한곳에 모이게 함으로써 클라이언트에서 직접 객체를 생성하지 않아도 되므로 코드를 보다 효율적으로 관리할 수 있게 됩니다.

하지만 설계에 따라 서브 클래스가 많이 생성될 수 있으므로 코드 복잡도는 올라갈 수 있습니다. 따라서 다른 패턴들과 마찬가지로 올바른 설계가 필요합니다.

### **참조**
> [https://refactoring.guru/design-patterns/factory-method](https://refactoring.guru/design-patterns/factory-method)
