---
title: 빌더 패턴 (Builder Pattern)
author: keencho
date: 2021-09-04 20:12:00 +0900
categories: [Design Pattern]
tags: [Design Pattern, Java]
---

# **빌더 패턴**
빌더 패턴은 복합 객체의 생성 과정과 표현 방법을 분리하여 동일한 생성 절차에서 서로 다른 표현 결과를 만들수 있게 하는 패턴입니다. 객체를 생성할때 흔하게 사용됩니다.

<br/>
## **구조**
![structure](/assets/img/custom/design-pattern/builder/structure.png)

1. 빌더 인터페이스는 모든 유형의 빌더에 사용되는 공통적인 단계를 선언합니다.
2. 빌더 구현체는 생성 단계에서 서로 다른 구현체를 제공합니다. 이 구현체는 공통 인터페이스를 따르지 않는 다른 빌더를 만들수도 있습니다.
3. Product1, Product2는 결과 객체입니다. 이들은 동일한 클래스 계층 구조나 인터페이스에 속할 필요가 없습니다.
4. Director 클래스는 개발자가 특정 Product의 구성을 생성하고 재사용할 수 있도록 생성 단계를 호출하는 순서를 지정합니다.
5. 클라이언트 클래스는 Director 클래스와 함께 빌더 객체중 하나와 연결되어야 합니다.

<br/>
## **적용 가능한 경우**
1. `점층적 생성자` 를 제거하고자 하는 경우 빌더 패턴을 사용하세요.

> 점층적 생성자란 아래 예제 코드와 같이 **비필수 매개변수가 없는 생성자부터 필수 매개변수를 전부 받는 생성자까지 생성자의 매개변수를 점층적으로 늘려가는 패턴을 뜻합니다.**
> 불필요한 배개변수까지 값을 지정해야 하며 생성자 수가 많아질수 있다는 단점을 가지고 있습니다.

```java
public class TelescopingConstructor {
    private String name;
    private String age;
    private String gender;
    private String job;
    private String birthday;
    private String address;

    public TelescopingConstructor(String name, String age) {
        this(name, age, "unknown");
    }

    public TelescopingConstructor(String name, String age, String gender) {
        this(name, age, gender, "unknown");
    }

    public TelescopingConstructor(String name, String age, String gender, String job) {
        this(name, age, gender, job, "unknown");
    }

    public TelescopingConstructor(String name, String age, String gender, String job, String birthday) {
        this(name, age, gender, job, birthday, "unknown");
    }

    public TelescopingConstructor(String name, String age, String gender, String job, String birthday, String address) {
        this.name = name;
        this.age = age;
        this.gender = gender;
        this.job = job;
        this.birthday = birthday;
        this.address = address;
    }
}
```
<span>2. </span>당신의 코드를 통해 상품을 다른 표현 방식으로 만들고 싶을때 빌더 패턴을 사용해보세요.
<span>3. </span>복잡한 클래스나 서로다른 트리들을 결합하여 객체를 생성하고자 할때 빌더 패턴을 사용해보세요.

<br/>
## **장단점**
#### **장점**
1. 객체를 단계별로 구성하거나 구성 단계를 지연하거나 재귀적으로 생성할수 있습니다.
2. 제품을 다양한 방식으로 표현할때 동일한 생성 코드를 재사용할 수 있습니다.
3. 복잡한 구성의 코드를 비즈니스 로직으로 분리할수 있음으로써 단일 책임 원칙을 만족합니다.

#### **단점**
1. 패턴을 적용시키고자 할때 N개의 새로운 클래스르 만들어야 하므로 코드의 복잡성이 증가합니다.
2. 생성비용 자체는 크지 않지만 성능이 중요시되는 상황이 오면 문제가 될수 있습니다.

<br/>
## **vs Java Beans Pattern**
빈 객체 생성후 setter 메소드를 이용한 값의 주입을 자바 빈 패턴이라고 합니다. 이는 실무에서도 자주 볼수있는 패턴입니다.

하지만 객체 생성 시점에 모든 값들을 주입하지 않아 어느 시점에 객체의 값이 변경될지 알수 없어 유지보수시 장애물이 될수 있습니다. 또 일일히 setter 메소드를 정의해줘야 한다는 단점도 있습니다. (~~롬복이라는 훌륭한 라이브러리가 있긴 하지요...~~)

이런 저런 이유로 객체를 생성하는 시점에 모든 값을 주입하여 불변 객체를 생성하는 빌더 패턴이 더 객체지향적인 패턴이라고 할수 있겠습니다.


<br/>
## **예제**
사람을 빌더 패턴을 이용해 만드는 예제입니다. 필수 정보는 Builder() 생성자에 포함되어야 합니다.

```java
public class Person {
    private String name;
    private String age;
    private String gender;
    private String job;
    private String birthday;
    private String address;

    public static class Builder {

        // 필수
        private String name;
        private String age;

        // 비필수수
        private String gender;
        private String job;
        private String birthday;
        private String address;

        public Builder(String name, String age) {
            this.name = name;
            this.age = age;
        }

        public Builder gender(String gender) {
            this.gender = gender;
            return this;
        }

        public Builder job(String job) {
            this.job = job;
            return this;
        }

        public Builder birthday(String birthday) {
            this.birthday = birthday;
            return this;
        }

        public Builder address(String address) {
            this.address = address;
            return this;
        }

        public Person build() {
            return new Person(this);
        }
    }

    public Person(Builder builder) {
        this.name = builder.name;
        this.age = builder.age;
        this.gender = builder.gender;
        this.job = builder.gender;
        this.birthday = builder.birthday;
        this.address = builder.address;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age='" + age + '\'' +
                ", gender='" + gender + '\'' +
                ", job='" + job + '\'' +
                ", birthday='" + birthday + '\'' +
                ", address='" + address + '\'' +
                '}';
    }
}
```

클라이언트 코드와 결과입니다. 값을 지정하지 않은 필드는 null 로 출력되는것을 확인할 수 있습니다.
```java
public class BuilderClient {
    public static void main(String[] args) {
        Person person = new Person
                .Builder("keencho", "20")
                .gender("man")
                .job("developer")
                .build();

        System.out.println(person.toString());

    }
}
```
```
Person{name='keencho', age='20', gender='man', job='man', birthday='null', address='null'}
```
간단한 빌더 패턴 예제입니다. 여기서는 메소드를 하나하나 만들었지만 저는 주로 롬복 라이브러리의 @Builder 어노테이션을 많이 사용하는 편입니다.

<br/>
### **결론**
빌더 패턴은 생성 패턴중의 하나로 자바빈 패턴의 단점을 보완하는 패턴입니다. 많이 사용되는 패턴중 하나이지만 성능에 민감한 상황이 되면 문제가 될수 있고 클래스는 시간이 갈수록 커지는 경향이 있으므로 주의하여 사용해야 합니다.

<br/>
### **참조**
> [https://refactoring.guru/design-patterns/builder](https://refactoring.guru/design-patterns/builder)

