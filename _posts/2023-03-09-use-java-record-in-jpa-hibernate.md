---
title: JPA 에서 Java Record 사용하기
author: keencho
date: 2023-03-09 20:12:00 +0900
categories: [JPA]
tags: [JPA, Hibernate]
---

# JPA 에서 Java Record 사용하기

## Java Record  
자바 레코드는 JDK14에서 preview로써 처음 등장했으며 JDK16에서 정식 스펙으로 포함된 기능입니다. 이는 클래스의 기능과 데이터 구조를 결합한 새로운 클래스 유형입니다.  

레코드는 `record` 예약어를 사용합니다. 레코드의 구성요소들은 메서드의 매개변수 정의와 동일한 형태의 구문을 사용하여 정의되며 타입과 이름을 포합합니다.  

예를들어 좌표를 나타내는 레코드는 다음과 같이 정의할 수 있습니다.
```java
public record Coordinates(double x, double y) {
    ...
}
```  

두 좌표 (x, y)를 보유하는 불변 객체를 생성하며 기존 클래스 형태의 객체에서 일반적으로 정의한 상용어구 코드 (constructor, getter, toString, hashCode, equals)를 정의하지 않아도 기본 구현을 제공하므로 클래스보다 간결한 방법으로 객체를 정의할 수 있습니다.  

## Entity로 사용하기
아쉽게도 JPA에서는 Record를 Entity로 사용할 수 없습니다. JPA 사양을 만족하는 엔티티 객체는 다음과 같은 요구사항을 충족해야 하기 때문입니다.

1. `@Entity` 어노테이션이 선언되어야 합니다.  
2. 매개변수가 없는 생성자가 `public` 혹은 `protected`으로 선언되어 있어야 합니다. JPA 구현체들이 쿼리 결과를 매핑할때 객체를 인스턴스화 할수 있어야 하기 때문입니다.  
3. JPA 구현체가 프록시 객체를 생성해야 하기 때문에 final로 선언되지 않아야 합니다.  
4. 엔티티 객체임을 확인하기 위해 하나 혹은 그 이상의 필드를 선언해야 합니다.
5. 필드를 db 컬럼으로 매핑하기 위해서 필드는 final이 아니어야 합니다.  
6. 필드에 접근하기 위해 getter와 setter를 제공해야 합니다.  

레코드는 매개 변수가 없는 생성자를 지원하지 않으며 클래스 자체가 final 클래스입니다. 레코드의 필드들 또한 final이며, 접근 메소드들 또한 요구사항에 적합한 네이밍 전략을 따르지 않습니다. 이러한 이유들 때문에 아직까지 JPA 에서는 Record를 Entity로 사용할 수 없습니다.  

## DTO로 사용하기  
조회한 정보를 변경하지 않는다면 DTO는 최고의 선택입니다. 이는 엔티티를 직접 조회하는 것보다 성능이 뛰어나며 도메인 모델과 API를 분리할 수 있습니다.  

JPA의 `Criteriabuilder.construct` 메소드를 사용하여 `CriteriaQuery`에서 생성자 호출을 정의할 수 있습니다.  

```java
public record OrderRecord(
        String id,
        OrderStatus status,
        String fromAddress,
        String fromName,
        String fromPhoneNumber,
        String toAddress,
        String toName,
        String toPhoneNumber,
        String itemName,
        int itemPrice,
        LocalDateTime createdDateTime
) { }
```  

```java
@Test
public void recordDTO() {
    var cb = entityManager.getCriteriaBuilder();
    var cq = cb.createQuery(OrderRecord.class);
    var root = cq.from(Order.class);

    cq.select(
            cb.construct(
                OrderRecord.class,
                root.get(Order_.id),
                root.get(Order_.status),
                root.get(Order_.fromAddress),
                root.get(Order_.fromName),
                root.get(Order_.fromPhoneNumber),
                root.get(Order_.toAddress),
                root.get(Order_.toName),
                root.get(Order_.toPhoneNumber),
                root.get(Order_.itemName),
                root.get(Order_.itemPrice),
                root.get(Order_.createdDateTime)
            )
    );

    cq.where(cb.notEqual(root.get(Order_.status), OrderStatus.FAILED));

    var q = entityManager.createQuery(cq);
    var list = q.getResultList();
}
```  

## 임베디드 객체로 사용하기  
### Hibernate < 6.0
Hiberante 6.0 이전 버전에서는 `record` 객체를 임베디드 객체로 사용할 수 없습니다. 이유는 위의 `record` 객체를 entity로 사용할 수 없는 이유와 같습니다.  

### Hibernate >= 6.0
Hibernate는 6 버전에서 `EmbeddableInstantiator` 기능을 소개하였습니다. 이 기능을 통해 enbeddable 객체를 보다 유연하게 인스턴스화 할수 있게 되었습니다. 이에대한 사이드 이펙트로 `record` 객체를 입데디드 객체로 사용할 수 있게 되었습니다.  

```java
@Entity
@Table(name = "order_new")
public class Order {
    ...
    
    @Embedded
    ShippingInfo fromInfo;

    @Embedded
    ShippingInfo toInfo;
    
    ...
}
```  

```java
@Embeddable
public record ShippingInfo(
        String name,
        String address,
        String number
) { }
```

`Order` 엔티티에 `fromInfo`와 `toInfo` 라는 배송지 정보를 임베디드 객체로써 선언하였습니다. 

> :warning: 위 예제가 동작하려면 네이밍 전략이 `ImplicitNamingStrategyComponentPathImpl` 이어야 합니다. (\<property name="hibernate.implicit_naming_strategy" value="org.hibernate.boot.model.naming.ImplicitNamingStrategyComponentPathImpl" /\>)  

`ShippingInfo` 객체를 임베디드 객체로써 사용하기 위해 이를 하이버네이트에 알려야 합니다. 이를 위해 `EmbeddableInstatiator`을 커스텀 하겠습니다.  

```java
public class ShippingInfoInstantiator implements EmbeddableInstantiator {
    @Override
    public Object instantiate(ValueAccess valueAccess, SessionFactoryImplementor sessionFactory) {
        var name = valueAccess.getValue(0, String.class);
        var address = valueAccess.getValue(1, String.class);
        var street = valueAccess.getValue(2, String.class);

        return new ShippingInfo(name, address, street);
    }

    @Override
    public boolean isInstance(Object object, SessionFactoryImplementor sessionFactory) {
        return object instanceof ShippingInfo;
    }

    @Override
    public boolean isSameClass(Object object, SessionFactoryImplementor sessionFactory) {
        return object.getClass().equals(ShippingInfo.class);
    }
}
```  

위 간단한 클래스에서 가장 중요한 것은 `instantiate` 메소드 입니다. 하이버네이트는 객체의 모든 필드 정보를 알파벳 순서로 담고 있는 `ValueAccess` 객체를 호출합니다. 
`ValudeAccess`객체에 인덱스로 접근하여 값을 원하는 유형으로 캐스팅 할 수 있습니다. 객체 인스턴스화에 필요한 모든 매개 변수를 추출한 후 마지막에 모든 매개 변수를 담고 있는 생성자를 생성하기 때문에 record를 임베디드 객체로써 사용할 수 있게 되는 것입니다.  

```java
@Embeddable
@EmbeddableInstantiator(ShippingInfoInstantiator.class)
public record ShippingInfo(
        String name,
        String address,
        String number
) { }
``` 

위 코드처럼 `@EmbeddableInstantiator` 어노테이션을 통해 커스텀한 `EmbeddableInstatiator` 클래스를 지정하기만 하면 끝입니다.  

이제 이 `record` 임베디드 객체가 잘 동작하는지 조회해 보도록 하겠습니다.  

```java
public record OrderRecord(
        String id,
        OrderStatus status,
        ShippingInfo fromShippingInfo,
        ShippingInfo toShippingInfo,
        String itemName,
        int itemPrice,
        LocalDateTime createdDateTime
) { }
```  

결과를 반환받기 위한 dto 객체입니다.  

```java
@Test
public void recordDTO() {
    var cb = entityManager.getCriteriaBuilder();
    var cq = cb.createQuery(OrderRecord.class);
    var root = cq.from(Order.class);

    cq.select(
            cb.construct(
                OrderRecord.class,
                root.get(Order_.id),
                root.get(Order_.status),
                root.get("fromInfo"),
                root.get("toInfo"),
                root.get(Order_.itemName),
                root.get(Order_.itemPrice),
                root.get(Order_.createdDateTime)
            )
    );

    cq.where(cb.like(root.get("fromInfo").get("name"), "%김%"));

    var q = entityManager.createQuery(cq);
    var list = q.getResultList();
}
```
테스트 코드입니다. record 객체에는 setter 메소드를 작성할 수 없어 `jpamodelgen` 프로세서가 접근할 수 없습니다. 따라서 부득이하게 string 형태로 임베디드 객체에 접근하였습니다.  

```sql
select
        o1_0.id,
        o1_0.status,
        o1_0.fromInfo_address,
        o1_0.fromInfo_name,
        o1_0.fromInfo_number,
        o1_0.toInfo_address,
        o1_0.toInfo_name,
        o1_0.toInfo_number,
        o1_0.itemName,
        o1_0.itemPrice,
        o1_0.createdDateTime 
    from
        order_new o1_0 
    where
        o1_0.fromInfo_name like ? escape ''
```  
정상적으로 sql문이 생성되었음을 확인할 수 있습니다.  

### Hibernate >= 6.2  
Hibernate 6.2 버전부터는 커스텀 `EmbeddableInstatiator` 클래스조차 필요 없게 되었습니다. `record` 객체에 `@Embeddable` 객체를 붙이는 것만으로도 record 객체를 임베디드 객체로 사용할 수 있습니다.  

```java
@Embeddable
public record ShippingInfo(
        String name,
        String address,
        String number
) { }
```  

물론 기존과 같이 커스텀 `EmbeddableInstatiator` 를 만들어 지정할 수도 있습니다.
