---
title: Hibernate @Where
author: keencho
date: 2022-05-26 08:12:00 +0900
categories: [Hibernate]
tags: [Hibernate, JPA]
---

# Hibernate @Where

### @Where 란?
떄때로 커스텀한 SQL을 이용해 엔티티 혹은 컬렉션을 필터링하고 싶은 경우가 있습니다. 이 경우 Hibernate 에서 제공하는 어노테이션중 `@Where` 이라는 어노테이션을 사용해 쉽게 필터링 할 수 있습니다. 

```java
@Target({TYPE, METHOD, FIELD})
@Retention(RUNTIME)
public @interface Where {
	String clause();
}
```

어노테이션 자체도 간단합니다. where 절을 clause 멤버변수에 작성하라고 안내하고 있습니다.  

### Entity에 적용하기
간단한 예제 엔티티를 작성해 보겠습니다. 부모 엔티티 (MainOrder)와 자식 엔티티(SubOrder)가 존재하며, 양방향으로 매핑을 해주도록 하겠습니다.  

```java
@Entity
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString(exclude = {"activeSubOrderList", "deActivatedSubOrderList", "fromNameLikeCList"})
@Table(name = "main_order")
public class MainOrder {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private LocalDate orderDate = LocalDate.now();

    @OneToMany(mappedBy = "mainOrder", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @Where(clause = "active = true")
    private List<SubOrder> activeSubOrderList;

    @OneToMany(mappedBy = "mainOrder", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @Where(clause = "active = false")
    private List<SubOrder> deActivatedSubOrderList;

    @OneToMany(mappedBy = "mainOrder", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @Where(clause = "LOWER(from_name) LIKE 'c%'")
    private List<SubOrder> fromNameLikeCList;

}
```

```java
@Entity
@Data
@AllArgsConstructor
@NoArgsConstructor
@Table(name = "sub_order")
public class SubOrder {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String fromName;
    private String fromPhoneNumber;
    private String fromAddress;

    private String toName;
    private String toPhoneNumber;
    private String toAddress;

    @ManyToOne(optional = false, fetch = FetchType.LAZY)
    private MainOrder mainOrder;

    private boolean active;

}
```  

`@Where` 어노테이션을 통해 각각의 조건에 부합하는 자식 엔티티 리스트를 가져올수 있도록 하였습니다.  

> `@Where` 어노테이션의 sql 문의 필드명은 데이터베이스의 필드명이어야 합니다.   
> 예를들어 위 MainOrder 엔티티의 fromNameLikeCList 의 경우 "LOWER(fromName) LIKE 'c%'" 라고 작성하면 에러가 발생합니다.

### 조회 결과 확인하기  
엔티티 객체를 작성하였으니 이제 더미데이터를 넣고 테스트를 수행해보겠습니다. 더미데이터는 [java-faker](https://github.com/DiUS/java-faker) 라이브러리를 사용하였습니다.  

```java
public void initData() {
    var mainOrderList = new ArrayList<MainOrder>();

    for (int i = 0; i < 5; i ++) {
        var mainOrder = new MainOrder();
        mainOrderList.add(mainOrderRepository.save(mainOrder));
    }

    for (int i = 0; i < 100; i ++) {
        var subOrder = new SubOrder();
        subOrder.setActive(i % 3 == 0);

        subOrder.setFromName(JavaFakerGenerator.getName());
        subOrder.setFromPhoneNumber(JavaFakerGenerator.getPhoneNumber());
        subOrder.setFromAddress(JavaFakerGenerator.getAddress());

        subOrder.setToName(JavaFakerGenerator.getName());
        subOrder.setToPhoneNumber(JavaFakerGenerator.getPhoneNumber());
        subOrder.setToAddress(JavaFakerGenerator.getAddress());

        subOrder.setMainOrder(mainOrderList.get(i % 5));
        subOrderRepository.save(subOrder);
    }
}
```  

활성:비활성의 비율을 약 1:2 정도의 비율로 세팅하여 테스트 데이터를 생성하였습니다.

```java
@Test
@Transactional
void whereAnnotationTest1() {
    var mainOrderList = mainOrderRepository.findAll();

    mainOrderList.forEach(mainOrder -> {
        mainOrder.getActiveSubOrderList();
        mainOrder.getDeActivatedSubOrderList();
        mainOrder.getFromNameLikeCList();
    });
}
```  

테스트 코드입니다. 테스트를 수행한다기 보다 쿼리를 확인할 목적으로 작성하였습니다.  

```sql
select
        activesubo0_.main_order_id as main_ord9_2_0_,
        activesubo0_.id as id1_2_0_,
        activesubo0_.id as id1_2_1_,
        activesubo0_.active as active2_2_1_,
        activesubo0_.from_address as from_add3_2_1_,
        activesubo0_.from_name as from_nam4_2_1_,
        activesubo0_.from_phone_number as from_pho5_2_1_,
        activesubo0_.main_order_id as main_ord9_2_1_,
        activesubo0_.to_address as to_addre6_2_1_,
        activesubo0_.to_name as to_name7_2_1_,
        activesubo0_.to_phone_number as to_phone8_2_1_ 
    from
        sub_order activesubo0_ 
    where
        (
            activesubo0_.active = true
        ) 
        and activesubo0_.main_order_id=1
```  

```sql
select
        deactivate0_.main_order_id as main_ord9_2_0_,
        deactivate0_.id as id1_2_0_,
        deactivate0_.id as id1_2_1_,
        deactivate0_.active as active2_2_1_,
        deactivate0_.from_address as from_add3_2_1_,
        deactivate0_.from_name as from_nam4_2_1_,
        deactivate0_.from_phone_number as from_pho5_2_1_,
        deactivate0_.main_order_id as main_ord9_2_1_,
        deactivate0_.to_address as to_addre6_2_1_,
        deactivate0_.to_name as to_name7_2_1_,
        deactivate0_.to_phone_number as to_phone8_2_1_ 
    from
        sub_order deactivate0_ 
    where
        (
            deactivate0_.active = false
        ) 
        and deactivate0_.main_order_id=1
```  

```sql
select
        fromnameli0_.main_order_id as main_ord9_2_0_,
        fromnameli0_.id as id1_2_0_,
        fromnameli0_.id as id1_2_1_,
        fromnameli0_.active as active2_2_1_,
        fromnameli0_.from_address as from_add3_2_1_,
        fromnameli0_.from_name as from_nam4_2_1_,
        fromnameli0_.from_phone_number as from_pho5_2_1_,
        fromnameli0_.main_order_id as main_ord9_2_1_,
        fromnameli0_.to_address as to_addre6_2_1_,
        fromnameli0_.to_name as to_name7_2_1_,
        fromnameli0_.to_phone_number as to_phone8_2_1_ 
    from
        sub_order fromnameli0_ 
    where
        (
            LOWER(fromnameli0_.from_name) LIKE 'c%'
        ) 
        and fromnameli0_.main_order_id=1
```  

세 쿼리를 보니 모두 `@Where` 어노테이션에 작성한 sql문 대로 잘 수행된것을 확인할 수 있습니다. 

### QueryDSL 에서 사용하기
QueryDSL 에서도 쉽게 사용할 수 있습니다. 예를들어 fromName이 c로 시작하는 SubOrder가 존재하는 경우에만 MainOrder를 가져와야 한다고 가정해보겠습니다.  

평소라면 where절에 직접 서브쿼리를 작성해야 하겠지만, `@Where` 어노테이션을 사용하면 쉽게 표현할 수 있습니다. 다음은 예제 / 테스트 코드입니다.  

```java
@Test
@Transactional
void whereAnnotationTest2() {

    var mq = QMainOrder.mainOrder;
    var sq = QSubOrder.subOrder;

    var subQueryFetch = jpaQueryFactory
            .select(mq)
            .from(mq)
            .where(JPAExpressions.select(sq.mainOrder.count()).from(sq).where(mq.eq(sq.mainOrder).and(sq.fromName.startsWithIgnoreCase("c"))).gt(0L))
            .fetch();

    var whereAnnotationFetch = jpaQueryFactory
            .select(mq)
            .from(mq)
            .where(mq.fromNameLikeCList.size().gt(0L))
            .fetch();

    Assert.isTrue(subQueryFetch.size() == whereAnnotationFetch.size(), "test failed");
}
```  

첫번째 `subQueryFetch`의 경우 where절에 직접 서브쿼리를 작성하였고, 두번째 `whereAnnotationFetch`의 경우 단순히 `@OneToMany`로 매핑된 엔티티의 사이즈를 불러와 비교하는 방식을 사용하였습니다.  

```sql
select
        mainorder0_.id as id1_0_,
        mainorder0_.order_date as order_da2_0_ 
    from
        main_order mainorder0_ 
    where
        (
            select
                count(suborder1_.main_order_id) 
            from
                sub_order suborder1_ cross 
            join
                main_order mainorder2_ 
            where
                suborder1_.main_order_id=mainorder2_.id 
                and mainorder0_.id=suborder1_.main_order_id 
                and (
                    lower(suborder1_.from_name) like 'c%' escape '!'
                )
        )>0
```  

```sql
select
        mainorder0_.id as id1_0_,
        mainorder0_.order_date as order_da2_0_ 
    from
        main_order mainorder0_ 
    where
        (
            select
                count(fromnameli1_.main_order_id) 
            from
                sub_order fromnameli1_ 
            where
                mainorder0_.id = fromnameli1_.main_order_id 
                and (
                    LOWER(fromnameli1_.from_name) LIKE 'c%'
                ) 
        )>0
```  

테스트도 통과하고 결과도 동일한것을 확인할 수 있습니다. 이처럼 QueryDSL에도 쉽게 사용가능한 것을 확인할 수 있습니다.
