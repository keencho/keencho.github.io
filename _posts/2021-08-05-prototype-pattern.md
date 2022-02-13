---
title: 프로토타입 패턴 (Prototype Pattern)
author: keencho
date: 2021-08-05 20:12:00 +0900
categories: [Design Pattern]
tags: [Design Pattern, Java]
---

# **프로토타입 패턴**
프로토타입 패턴은 원형이 되는 인스턴스를 사용해 새롭게 생성할 객체의 종류를 명시하여 새로운 객체가 생성될 시점에 인스턴스의 타입이 결정되도록 하는 패턴입니다.

## **구조**
![structure](/assets/img/custom/design-pattern/prototype/structure.png)

1. 임의의 인스턴스를 복제하는 메소드를 가진 인터페이스인 `Prototype` 인터페이스를 생성합니다. 대두분의 경우 이 인터페이스에는 `clone()` 메소드 하나만 선언되어 있습니다.
2. `Prototype` 인터페이스를 구현하는 클래스를 생성합니다. 이 클래스는 원본 객체의 데이터를 복사하는것 말고도 연결된 객체의 복사와 관련된 작업이나 전의 의존성에서 벗어나게 하는 작업 등을 수행할 수 있습니다.
3. 클라이언트는 `Prototype` 인터페이스를 따르는 모든 객체를 복사하여 인스턴스를 생성할 수 있습니다.

## **적용 가능한 경우**
1. 코드가 복사해야 하는 구현 클래스에 의존하지 않아야 하는 경우 프로토타입 패턴을 사용할 수 있습니다.
  - 이 경우는 코드가 인터페이스를 통해 써드파티 코드와 함께 작동할 경우 많이 발생합니다.
2. 객체를 초기화 하는 방식만 다를뿐 서브클래스의 수를 줄이려는 경우 프로토타입 패넡을 사용할 수 있습니다.

## **장단점**
#### **장점**
1. 구현 클래스에 직접 연결하지 않고 객체를 복사할 수 있습니다.
2. 프로토타입이 미리 정의되어 있기 때문에 중복되는 초기화 코드를 제거할 수 있습니다.
3. 복잡한 오브젝트를 보다 편리하게 만들수 있습니다.

#### **단점**
1. 순환 참조가 있는 복잡한 객체를 복제하는 것은 매우 까다로울 수 있습니다.

## **예제**
db에서 모든 자동차의 종류를 가져와 클라이언트에 보여줘야하는 프로그램이 있다고 가정해 보겠습니다. 아래는 간단한 Car 클래스 입니다.

```java
public class Car implements Cloneable {

    private List<String> carList;

    public Car() {
        this.carList = new ArrayList<>();
    }

    public Car(List<String> carList) {
        this.carList = carList;
    }

    public List<String> getCarList() {
        return carList;
    }

    public void listAll() {
        // db에서 자동차의 종류를 가져오는 코드
        this.carList.add("truck");
        this.carList.add("suv");
        this.carList.add("sedan");
        this.carList.add("sports car");
    }

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return new Car(new ArrayList<>(this.carList));
    }
}
```
`Cloneable` 인터페이스를 구현하여 `Car` 클래스를 생성하였고 `clone()` 메소드를 통해 깊은 복사 방식으로 인스턴스를 복제하도록 하였습니다.

```java
public class CarClient {
    public static void main(String[] args) throws CloneNotSupportedException {
        Car car = new Car();
        car.listAll();

        Car car2 = (Car) car.clone();
        Car car3 = (Car) car.clone();

        List<String> car2List = car2.getCarList();
        List<String> car3List = car3.getCarList();

        car2List.add("mini car");
        car3List.remove("truck");

        System.out.println(car.getCarList());
        System.out.println(car2List);
        System.out.println(car3List);

    }
}
```
```
[truck, suv, sedan, sports car]
[truck, suv, sedan, sports car, mini car]
[suv, sedan, sports car]
```
위 코드는 자동차 클라이언트와 클라이언트의 실행 결과 입니다. 깊은 복사 방식을 사용했으므로 각각의 객체 인스턴스가 모두 다른 결과를 도출해 내는 것을 확인할수 있습니다.

### **결론**
프로토타입 패턴은 db에서 빈번하게 데이터를 가져올때, 그 데이터가 항상 똑같은 값을 반환하는 경우 유용하게 사용할 수 있습니다.
db에서 그때그때 가져오는 것이 좋은것 아니야? 라고 생각하실수 있겠지만 db에 접근하는데 사용하는 자원, 비용이 객체를 복사하는 비용보다 훨씬 크다는것을 염두해 두셔야 합니다.

### **참조**
> [https://refactoring.guru/design-patterns/prototype](https://refactoring.guru/design-patterns/prototype)

