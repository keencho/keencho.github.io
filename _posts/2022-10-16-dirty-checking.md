---
title: Dirty Checking
author: keencho
date: 2022-10-16 07:12:00 +0900
categories: [Hibernate]
tags: [JPA, Hibernate]
---

# **Dirty Checking**

## **1. 개요**
하이버네이트 / jpa를 사용하다보면 이미 저장된 엔티티의 경우 jpa repository의 save() 메소드를 호출하지 않아도 한 트랜잭션이 종료되면 엔티티의 변경점이 자동으로 업데이트되는 현상을 발견할 수 있습니다.

```java
var delivery = deliveryRepository.findById(1L).orElseThrow(() -> new RuntimeException("delivery entity not exist!"));
delivery.setDtDeliveryStartedAt(LocalDateTime.now());
```

위와같이 save 메소드 없이 코드를 작성해도 트랜잭션이 종료되면 `dtDeliveryStartedAt` 컬럼이 업데이트되게 됩니다.

위 동작을 Dirty Checking 이라 하는데요, Dirty Checking의 동작 메커니즘은 다음과 같습니다.

> 영속선 컨텍스트는 플러시가 일어날 경우 엔티티의 변경점들을 대기열에 넣습니다.
> 하이버네이트는 관리되는 엔티티들의 모든 변경을 자동으로 감지하고 개발자들을 대신하여 SQL Update문을 계획하고 수행합니다.

하이버네이트는 기본적으로 모든 엔티티의 속성 / 변경점을 확인합니다. 엔티티가 로딩될 때마다 하이버네이트는 모든 엔티티 속성값을 가지고 있는 스냅샷(복사본)을 생성합니다. 그 후 플러시가 일어나는 시점에 엔티티와 스냅샷을 비교하여 변경된 부분을 체크하고 업데이트 합니다.

단순히 불러와서 값을 변경했다하여 모두 업데이트되는 것이 아닙니다. Dirty Checking이 일어나려면 아래 조건을 충족해야 합니다.

1. 엔티티가 영속상태인 경우
2. Transaction 안에서 엔티티의 값을 변경하는 경우
3. 이미 저장된 엔티티인 경우

스프링을 사용한다면 `@Transactional` 어노테이션을 서비스 레이어에서 사용하면 위 조건은 만족하게 됩니다.

## **2. 특정 컬럼만 업데이트하기**
JPA에서는 모든 필드를 업데이트하는 방식을 기본값으로 사용합니다. 이는 다음과 같은 장점을 갖고 있습니다.

1. 생성되는 쿼리가 모두 같아 어플리케이션이 실행되는 시점에 만들어서 재사용 가능
2. 업데이트되는 컬럼수가 동일하기 때문에 캐싱된 SQL 구문을 사용할 수 있고, 데이터베이스 입장에서 업데이트 컬럼이 작다면 특정 컬럼만 업데이트하는 구문에 비해 성능상 이점을 가져갈 수 있음

특정 컬럼만 업데이트하는 방법들을 몇가지 소개하긴 할테지만, 특정 상황이 아닌 경우 오히려 이러한 방법들은 전체 컬럼을 업데이트하는 방법보다 오히려 많은 자원을 소모할 수 있습니다.

컬럼이 30개정도 된다면 특정 컬럼만 업데이트 하는 것이 도움이 될테지만 애초에 컬럼이 30개라면 정규화가 잘못되어 있다는 의미겠죠? 항상 생각하는 것이지만 역시 제일 중요한 것은 db 설계 같습니다.

#### **1. @DynamicUpdate 사용**
첫번째 방법은 엔티티에 `@DynamicUpdate` 어노테이션을 선언하는 것입니다.

```java
@Entity
@Data
@NoArgsConstructor
@Table(name = "delivery")
@DynamicUpdate
public class Delivery {
  ...
}
```

위와같이 어노테이션을 선언하는 것만으로도 변경 필드만 반영되게 할 수 있습니다.

#### **2. @Query 사용**
두번째 방법은 `@Query`어노테이션을 사용하여 직접 update 구문을 작성하는 것입니다.

```java
@Modifying
@Query("UPDATE Delivery d set d.dtDeliveryStartedAt = :date where d.deliveryId = :id")
void updateDtDeliveryStartedAtById(@Param("date") LocalDateTime date, @Param("id") Long id);
```

이렇듯 직접 update 문을 작성하여 특정 컬럼만 업데이트 할 수 있습니다. select가 아닌 DML 이기 때문에 `@Modifying` 어노테이션을 붙여야 함을 잊지 마세요.

#### **3. QueryDSL JPAUpdateClause 사용**
세번째 제가 주로 사용하는 방법은 QueryDSL의 `JPAUpdateClause` 클래스를 사용하는 방법입니다.

```java
@Test
public void test() {
    EntityManager em = emf.createEntityManager();
    QDelivery q = QDelivery.delivery;
    JPAUpdateClause jpaUpdateClause = new JPAUpdateClause(em, q);
    Map<Path<?>, Object> map = new HashMap<>();

    map.put(q.dtDeliveryStartedAt, LocalDateTime.now());

    List<Path<?>> paths = new ArrayList<>();
    List<Object> values = new ArrayList<>();

    for (var entry : map.entrySet()) {
        paths.add(entry.getKey());
        values.add(entry.getValue());
    }

    long result = jpaUpdateClause.where(q.deliveryId.eq(1L)).set(paths, values).execute();

    Assert.isTrue(result == 1, "update result is not 1");
}
```

위와같이 `JPAUpdateClause` 를 사용하면 `@Query` 어노테이션을 사용하는 방식에 비해 조금더 type-safe하게 update 문을 자바 코드로 작성할 수 있습니다. 위 코드는 많은 보일러플레이트 코드를 만들수 있는데요, 제 경우 별도의 repository를 만들어서 코드를 작성하곤 합니다. ([링크](https://github.com/keencho/lib-spring/tree/master/src/main/java/com/keencho/lib/spring/jpa/querydsl/repository) 에서 확인해보세요.)

그렇다면 아래와 같이 코드량이 줄게 되지요.

```java
@Test
public void test() {
    var q = QDelivery.delivery;
    var map = Q.newUpdateMap();

    map.put(q.dtDeliveryStartedAt, LocalDateTime.now());

    deliveryRepository.updateOne(q.deliveryId.eq(1L), map);
}
```


