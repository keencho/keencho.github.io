---
title: QueryProjection 에서 한단계 더 나아가 QueryProjectionBuilder 만들기 (1)
author: keencho
date: 2022-09-28 07:12:00 +0900
categories: [QueryDSL]
tags: [Java, JPA, QueryDSL]
---

# **QueryProjection 에서 한단계 더 나아가 QueryProjectionBuilder 만들기 (1) - 개요**
Querydsl을 사용해 db조회를 하는 경우 엔티티 자체를 반환하는 경우는 잘 없습니다. 대부분 필요한 필드만 지정하여 `Projections`를 사용하여 조회하게 되는것 같습니다.

잘 알려진 projection 방법에는 3가지가 있습니다.
> 1. Projections.bean - getter / setter 필요
> 2. Projections.constructor - field 직접 접근
> 3. @QueryProjection - ConstructorExpression를 상속받은 Q 클래스 생성

첫번째 방식과 두번째 방식은 조회하려는 필드명이 일치해야 한다는 단점이 있습니다.
```java
public class SimpleDTO {
    Long deliveryId;
    Long orderId;
    String field;
}

...

var p = Projections.fields(SimpleDTO.class, q.deliveryId, q.order.orderId, q.fromAddress);
```

예를들어 위와같이 조회했을때 SimpleDTO의 field 와 entity q 객체의 fromAdress의 필드명이 일치하지 않기 때문에 결과를 조회했을때 null이 넘어오게 됩니다.

그래서 처음 querydsl을 사용하고 실무에 적용때는 @QueryProjection 방식을 사용했었습니다. Entity의 Q객체를 생성하는 것처럼 DTO Class의 Q객체를 생성하고, 생성한 Q 객체의 생성자를 사용하는 방식이기 때문에 조회하고자 하는 필드의 순서가 정확해야 한다는 조건이 있습니다.

또, 기존 코드와의 호환성을 유지하려면 오리지날 DTO Class에 필드를 추가할때마다 생성자를 하나씩 추가해야 한다는 문제가 생겼습니다.
```java
public class SimpleDTO {
    Long deliveryId;
    Long orderId;
    String field;

    @QueryProjection
    public SimpleDTO(Long deliveryId, Long orderId, String field) {
        this.deliveryId = deliveryId;
        this.orderId = orderId;
        this.field = field;
    }

    @QueryProjection
    public SimpleDTO(Long deliveryId) {
        this.deliveryId = deliveryId;
    }

    @QueryProjection
    public SimpleDTO(Long deliveryId, Long orderId) {
        this.deliveryId = deliveryId;
        this.orderId = orderId;
    }

    @QueryProjection
    public SimpleDTO(Long orderId, String field) {
        this.orderId = orderId;
        this.field = field;
    }
}
```

사실 위 문제는 어플리케이션을 설계할때 '이렇게 저렇게 하자~' 라고 미리 정해두고 가면 충분히 해결할수 있는 문제입니다. 하지만 실무에서는 기획이 바뀌는일이 허다하죠. 그래서 저는 저만의 방식으로 Projection을 만들어보기로 하였습니다.

## **초기의 생각**
처음에 생각했던 방식은 map을 만들고, 조회할 필드와 Expression을 map에 집어넣고 조회후에 다시 리플렉션으로 dto에 값을 꽂아넣는 방식이었습니다. 코드로 풀면 다음과 같습니다.

```java
public class SimpleDTO {
    Long deliveryId;
    Long orderId;
    String field;

    private static Map<String, Expression<?>> bindings;

    static {
        var q = Q.delivery;

        bindings.put("deliveryId", q.deliveryId);
        bindings.put("orderId", q.order.orderId);
        bindings.put("field", q.fromAddress);
    }
}

...

queryFactory.select(SimpleDTO.bindings).from(q).fetch();
```

이 방식은 querydsl에 그다지 의존적이지 않으나 컴파일시점에 오류를 잡을수 없다는 문제가 존재합니다. 저는 querydsl에 의존적이더라도 오류를 잡을수 있는 방식을 선호하기 때문에 위 방식은 선택하지 않았습니다.

## **QBuilder를 만들자!**
QueryProjection이 Q클래스를 생성하는 것처럼 builder, setter를 사용할 수 있는 저만의 Q클래스를 만들어보기로 하였습니다. 생성되는 Q 클래스의 모양새는 다음과 같습니다.

```java
import com.querydsl.core.types.dsl.*;
import com.keencho.lib.spring.jpa.querydsl.KcQBean;
/**
 * com.keencho.spring.jpa.querydsl.dto.KcQSimpleDTO is a KcQuerydsl Projection type for SimpleDTO
 */
@javax.annotation.processing.Generated("com.keencho.lib.spring.jpa.querydsl.KcProjectionSerializer")
public class KcQSimpleDTO extends KcQBean<SimpleDTO> {

    public KcQSimpleDTO() {
        super(SimpleDTO.class);
    }

    public KcQSimpleDTO(Builder builder) {
        super(SimpleDTO.class);
        this.deliveryDTO = builder.deliveryDTO;
        this.deliveryId = builder.deliveryId;
        this.field = builder.field;
        this.orderId = builder.orderId;
    }

    private static final long serialVersionUID = 1803879685L;

    private com.querydsl.core.types.Expression<com.keencho.spring.jpa.querydsl.dto.DeliveryDTO> deliveryDTO;

    public void setDeliveryDTO(com.querydsl.core.types.Expression<com.keencho.spring.jpa.querydsl.dto.DeliveryDTO> deliveryDTO) {
        this.deliveryDTO = deliveryDTO;
    }

    private com.querydsl.core.types.Expression<Long> deliveryId;

    public void setDeliveryId(com.querydsl.core.types.Expression<Long> deliveryId) {
        this.deliveryId = deliveryId;
    }

    private com.querydsl.core.types.Expression<String> field;

    public void setField(com.querydsl.core.types.Expression<String> field) {
        this.field = field;
    }

    private com.querydsl.core.types.Expression<Long> orderId;

    public void setOrderId(com.querydsl.core.types.Expression<Long> orderId) {
        this.orderId = orderId;
    }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {

        private com.querydsl.core.types.Expression<com.keencho.spring.jpa.querydsl.dto.DeliveryDTO> deliveryDTO;

        public Builder deliveryDTO(com.querydsl.core.types.Expression<com.keencho.spring.jpa.querydsl.dto.DeliveryDTO> deliveryDTO) {
            this.deliveryDTO = deliveryDTO;
            return this;
        }

        private com.querydsl.core.types.Expression<Long> deliveryId;

        public Builder deliveryId(com.querydsl.core.types.Expression<Long> deliveryId) {
            this.deliveryId = deliveryId;
            return this;
        }

        private com.querydsl.core.types.Expression<String> field;

        public Builder field(com.querydsl.core.types.Expression<String> field) {
            this.field = field;
            return this;
        }

        private com.querydsl.core.types.Expression<Long> orderId;

        public Builder orderId(com.querydsl.core.types.Expression<Long> orderId) {
            this.orderId = orderId;
            return this;
        }

        public KcQSimpleDTO build() {
            return new KcQSimpleDTO(this);
        }

    }

}
```

이렇게 되면 필드를 추가해도 기존의 코드에 영향을 주지 않고 런타임에 오류도 잡을수 있고 builder, setter를 사용할수 있게 됩니다.

조회할때는 이런식으로 조회할 수 있습니다. (아래 사용된 repository는 JPARepository를 상속받은 것이 아닌 제가 만든 repository 입니다.)
```java
@Test
public void queryTest() {
    var q = Q.delivery;

    var deliveryDTO = new KcQDeliveryDTO();
    deliveryDTO.setFromAddress(q.fromAddress);
    deliveryDTO.setFromName(q.fromName);
    deliveryDTO.setFromNumber(q.fromNumber);
    deliveryDTO.build();

    var simpleDTO = KcQSimpleDTO.builder()
            .orderId(q.order.orderId)
            .deliveryId(q.deliveryId)
            .field(q.fromAddress)
            .deliveryDTO(deliveryDTO.build())
            .build();

    var list = deliveryRepository.selectList(null, simpleDTO);

    System.out.println(list.size());
}
```

## **다음 포스팅에서...**
라이브러리 제작 과정 및 자세한 코드는 다음 포스팅에서 다루도록 하겠습니다.
