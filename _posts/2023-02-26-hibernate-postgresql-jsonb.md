---
title: Hibernate 에서 PostgreSQL JSONB 다루기
author: keencho
date: 2023-02-26 20:12:00 +0900
categories: [Hibernate]
tags: [Hibernate, PostgreSQL]
---

# **Hibernate 에서 PostgreSQL JSONB 다루기**
Hibnerate6 이전 버전에서 PostgreSQL의 JSONB 타입을 다루기 위해선 직접 `UserType` 인터페이스를 구현하거나 [이런 라이브러리](https://github.com/vladmihalcea/hypersistence-utils) 를 사용하여 JSONB 타입을 다뤘었습니다.

Hibernate6 부터는 이러한 기능을 표준으로 제공하기 시작하여 JSON 컬럼을 entity의 속성으로서 사용할 수 있게 되었습니다.

> :warning: 아래 내용은 Hibernate6 이상의 버전을 필요로 합니다. Hibernate6 이전 버전을 사용하는 경우 (Hibernate 5.x, Spring Boot 2.x, Spring Data JPA 2.x...) 그냥 `hypersistence-utils`를 사용하시길 권장드립니다.

## **@JdbcTypeCode**
`@JdbcTypeCode` 어노테이션을 필드에 붙여주기만 하면 Hibernate는 RDB의 종류에 따라 자동으로 JSON 컬럼을 정의합니다. PostgreSQL 경우엔 JSONB 타입이 되겠죠. 런타임에는 직렬화 / 역직렬화 할 수 있는 JSON 라이브러리를 사용하여 값을 컨트롤 합니다.

```java
@Data
public class ShippingInfo implements Serializable {
    private String name;
    private String number;
    private String address;
}

```

```java
@Entity
@Data
@Table(name = "order_new")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    Long id;

    @JdbcTypeCode(SqlTypes.JSON)
    ShippingInfo fromInfo;

    @JdbcTypeCode(SqlTypes.JSON)
    ShippingInfo toInfo;
}
```

```
[DEBUG] [main] SQL -
    create table order_new (
        id bigint not null,
        fromInfo jsonb,
        toInfo jsonb,
        primary key (id)
    )
```

## **UserType 인터페이스 구현**
`UserType` 인터페이스를 통해 직접 타입을 구현하는 방법도 있습니다. `UserType` 인터페이스는 Hibernate6 버전 이전에도 존재했었습니다. 다만 Hibernate6 버전으로 올라오면서 이 인터페이스는 제네릭을 사용하도록 변경되었습니다.

![hibernate5-usertype](/assets/img/custom/hibernate-postgresql-jsonb/hibernate5-usertype.JPG)

Hibernate5 - UserType
{: style="text-align: center;"}

![hibernate6-usertype](/assets/img/custom/hibernate-postgresql-jsonb/hibernate6-usertype.JPG)

Hibernate6 - UserType
{: style="text-align: center;"}

뿐만 아니라 기존에는 `@TypeDef` 어노테이션으로 사용할 JSON 타입 클래스를 먼저 등록하고 `@Type` 어노테이션을 사용했었어야 했다면, 이제 `@Type` 어노테이션만 사용하면 되도록 변경되었습니다. (`@TypeDef` 어노테이션은 삭제 되었습니다.)

```java
@Entity
@Data
@Table(name = "order_new")
@TypeDef(name = "ShippingInfoJsonType", typeClass = ShippingInfoJsonType.class)
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    Long id;

    @Column
    @Type(type = "ShippingInfoJsonType")
    ShippingInfo fromInfo;

    @Column
    @Type(type = "ShippingInfoJsonType")
    ShippingInfo toInfo;
}
```

```java
@Entity
@Data
@Table(name = "order_new")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    Long id;

    @Column
    @Type(ShippingInfoJsonType.class)
    ShippingInfo fromInfo;

    @Column
    @Type(ShippingInfoJsonType.class)
    ShippingInfo toInfo;
}
```

조금이나마 코드가 간결해지고 type-safe 하게 변경된 것을 확인할 수 있습니다.

사용법은 위와같고 `UserType` 인터페이스 구현 방법은 기존과 동일합니다. 필요한 메소드들을 제네릭 타입에 맞게 재정의 하면 됩니다.

```java
public class ShippingInfoJsonType implements UserType<ShippingInfo> {

    @Override
    public int getSqlType() {
        return SqlTypes.JSON;
    }

    @Override
    public Class<ShippingInfo> returnedClass() {
        return ShippingInfo.class;
    }

    ...
}
```

## **JSONB 컬럼의 필드 업데이트**
setter 메소드를 사용해 JSONB 컬럼의 필드값을 변경하면 트랜잭션 종료 이후 Hibernate에 의해 값이 업데이트 됩니다.

```java
private static void doInTransaction(Runnable task) {
    entityManager.getTransaction().begin();

    task.run();

    entityManager.getTransaction().commit();
}

...

doInTransaction(() -> {
    var order = entityManager.find(Order.class, 1L);
    order.getFromInfo().setNumber("01000001111");
});
```

sql 로그는 다음과 같습니다.

```
[DEBUG] [main] SQL -
    update
        order_new
    set
        fromInfo=?,
        toInfo=?
    where
        id=?
[TRACE] [main] bind - binding parameter [1] as [JSON] - [ShippingInfo(name=석설수, number=01000001111, address=서울특별시 구로구 개포로109길)]
[TRACE] [main] bind - binding parameter [2] as [JSON] - [ShippingInfo(name=성인건, number=01018657836, address=서울특별시 금천구 개포로34길)]
[TRACE] [main] bind - binding parameter [3] as [BIGINT] - [1]
```

id가 1인 엔티티를 찾고 `받는쪽 배송정보` JSONB 컬럼의 번호를 `01000001111` 로 변경하였습니다. 정상적으로 잘 업데이트 된 것을 확인할 수 있습니다.

## **JSONB 조건검색**
그렇다면 `받는쪽 배송정보` JSONB 컬럼의 성이 `김` 으로 시작하는 row를 검색하려면 어떻게 해야 할까요? sql로 표현하면 다음과 같습니다.

```sql
select *
from order_new o
where o.frominfo ->> 'name' like '김%';
```

### **Hibernate < 6.2**
Hibernate 6.2 이전 버전에서는 아쉽게도 JPQL로 위 sql을 표현할 수 없습니다. 어쩔수 없이 `createNativeQuery(query)` 메소드를 사용할 수 밖에 없습니다.

```java
entityManager.createNativeQuery("SELECT * FROM order_new o WHERE o.frominfo ->> 'name' LIKE :name ")
             .setParameter("name", "김%")
             .getResultList()
```

```
[DEBUG] [main] SQL -
    SELECT
        *
    FROM
        order_new o
    WHERE
        o.frominfo ->> 'name' LIKE ?
[TRACE] [main] bind - binding parameter [1] as [VARCHAR] - [김%]
```

### **Hibernate >= 6.2**
Hibernate 6.2 버전에서 드디어 JPQL로 JSONB 컬럼을 탐색할 수 있게 되었습니다.

```java
@Data
@Embeddable
public class ShippingInfo implements Serializable {
    private String name;
    private String number;
    private String address;
}
```

일단 `ShippingInfo` 클래스에 `@Embeddable` 어노테이션을 붙여 JPA가 이 클래스를 인식할 수 있도록 합니다.

```java
@Test
@DisplayName("JPQL JSONB 조건 테스트 (version >= 6.2")
public void jsonbJPQLCondition() {
    var orderList = entityManager
            .createQuery("SELECT o FROM Order o WHERE o.fromInfo.name LIKE :name", Order.class)
            .setParameter("name", "김%")
            .getResultList();

    Assertions.assertTrue(orderList.stream().allMatch(o -> o.getFromInfo().getName().startsWith("김")));
}
```

```
[DEBUG] [main] SQL -
    select
        o1_0.id,
        o1_0.fromInfo,
        o1_0.toInfo
    from
        order_new o1_0
    where
        cast(o1_0.fromInfo->>'name' as varchar(255)) like ? escape ''
[TRACE] [main] bind - binding parameter [1] as [VARCHAR] - [김%]
```

테스트 코드와 sql 로그입니다. Hibernate가 JPQL을 올바른 sql로 변환했네요.

### **QueryDSL**
JPQL로 JSON 컬럼을 탐색할 수 있다는 것은 QueryDSL에서도 사용할 수 있게 되었다는 것을 의미합니다.

이번에는 QueryDSL로 조금 복잡한 쿼리를 구현해 보도록 하겠습니다. 조건은 다음과 같습니다.

> 1. 보내는 분의 성은 `김`이 아니어야 함
> 2. 보내는 분의 성으로 결과를 그루핑함
> 3. 결과를 내림차순으로 정렬하여 어떤 성이 가장 많은지 확인

위 조건을 sql로 표현하면 다음과 같습니다.

```sql
select substr(o.frominfo ->> 'name', 1, 1),
       count(*)
from order_new o
where o.frominfo ->> 'name' not like '김%'
group by substr(o.frominfo ->> 'name', 1, 1)
order by count(*) desc;
```

실무에서 이런 요구사항이 던져진다면 '그냥 날쿼리 쓸까?' 라는 생각이 들곤 합니다. 그러나 이제는 QueryDSL로 이런 쿼리도 표현할 수 있게 되었습니다.

일단 조회결과로 반환될 dto를 만들어 줍니다.

```java
@Data
public class OrderAggregationDTO {
    private String lastName;
    private int count;

    @QueryProjection
    public OrderAggregationDTO(String lastName, int count) {
        this.lastName = lastName;
        this.count = count;
    }
}
```

다음으로 QueryDSL 코드를 작성합니다.

```java
public void jsonbQueryDSLCondition() {
    var q = QOrder.order;

    var lastName = q.fromInfo.name.substring(0, 1);
    var count = q.count();

    var list = jpaQueryFactory()
            .select(new QOrderAggregationDTO(lastName, count))
            .from(q)
            .where(q.fromInfo.name.startsWith("김").not())
            .groupBy(lastName)
            .orderBy(count.desc())
            .fetch();

    list.forEach(item -> System.out.printf("성: %s / 갯수: %d개%n", item.getLastName(), item.getCount()));
}
```

```
[DEBUG] [main] SQL -
    select
        substr(cast(o1_0.fromInfo->>'name' as varchar(255)),1,1),
        count(o1_0.id)
    from
        order_new o1_0
    where
        cast(o1_0.fromInfo->>'name' as varchar(255)) not like ? escape '!'
    group by
        substr(cast(o1_0.fromInfo->>'name' as varchar(255)),1,1)
    order by
        count(o1_0.id) desc
[TRACE] [main] bind - binding parameter [1] as [VARCHAR] - [김%]
```

```
성: 주 / 갯수: 6개
성: 변 / 갯수: 4개
성: 신 / 갯수: 4개
성: 진 / 갯수: 4개
성: 탁 / 갯수: 4개
성: 박 / 갯수: 3개
성: 조 / 갯수: 3개
성: 엄 / 갯수: 3개
성: 노 / 갯수: 3개
성: 오 / 갯수: 3개
성: 석 / 갯수: 2개
성: 홍 / 갯수: 2개
성: 차 / 갯수: 2개
성: 선 / 갯수: 2개
성: 심 / 갯수: 2개
성: 표 / 갯수: 2개
성: 우 / 갯수: 2개
성: 손 / 갯수: 2개
성: 남 / 갯수: 2개
성: 도 / 갯수: 2개
성: 소 / 갯수: 2개
성: 은 / 갯수: 2개
성: 허 / 갯수: 2개
성: 안 / 갯수: 2개
성: 전 / 갯수: 2개
성: 유 / 갯수: 2개
성: 이 / 갯수: 2개
성: 곽 / 갯수: 1개
성: 편 / 갯수: 1개
성: 한 / 갯수: 1개
성: 황 / 갯수: 1개
성: 권 / 갯수: 1개
성: 현 / 갯수: 1개
성: 용 / 갯수: 1개
성: 원 / 갯수: 1개
성: 성 / 갯수: 1개
```

sql 로그와 결과를 확인합니다. 이번에도 JPQL이 올바른 sql로 변환된 것을 확인할 수 있습니다.

# **결론**
Hibernate6 버전부터 쉽게 JSON 타입의 컬럼을 다를 수 있게 되었습니다. 뿐만 아니라 6.2 버전부터는 조회 또한 JPQL로 표현할 수 있게 되었는데요,
이를 통해 만약 QueryDSL을 사용한다면 `querydsl-sql`, `blaze-persistence` 를 사용하지 않고 `querydsl-jpa`만으로 type-safe한 쿼리를 작성할 수 있게 되었습니다.

또한 [CTE](https://in.relation.to/2023/02/20/hibernate-orm-62-ctes/) 와 [Partitioning](https://in.relation.to/2023/02/08/hibernate-orm-62-partitioning/) 등 새로운 기능들이 Hibernate 6.2 버전에서 선보여질 예정입니다.
JPA의 태생적인 한계를 Hibernate를 통해 하나 둘 해결할 수 있게 되었으니 앞으로는 개발자가 조금더 쉽고 간단하게 자바로 db를 컨트롤 할 수 있을것 같습니다.

