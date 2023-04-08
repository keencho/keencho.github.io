---
title: QueryProjection 에서 한단계 더 나아가 QueryProjectionBuilder 만들기 (2)
author: keencho
date: 2022-10-02 07:12:00 +0900
categories: [QueryDSL]
tags: [Java, JPA, QueryDSL]
---

# **QueryProjection 에서 한단계 더 나아가 QueryProjectionBuilder 만들기 (2) - 개발 과정, 코드**

> 이 포스팅에서 설명하는 코드는 한 프로젝트 내에서 사용할 수 없습니다.  
> 라이브러리 프로젝트 A, 실제 어플리케이션 프로젝트 B 로 나뉘어야 합니다.  
> java17, maven 을 사용합니다.  


## **1. @QueryProjection 으로 생성되는 클래스 분석**  
클래스를 프로젝션 타입으로 사용하기 위해 우리는 클래스 생성자에 @QueryProjection 어노테이션을 붙이고 Q 클래스를 생성합니다. 생성되는 Q 클래스는 다음과 같습니다.  

```java
/**
 * com.keencho.libormtest.model.QCustomerDTO is a Querydsl Projection type for CustomerDTO
 */
@Generated("com.querydsl.codegen.DefaultProjectionSerializer")
public class QCustomerDTO extends ConstructorExpression<CustomerDTO> {

    private static final long serialVersionUID = -785800583L;

    public QCustomerDTO(com.querydsl.core.types.Expression<Long> id, com.querydsl.core.types.Expression<String> loginId, com.querydsl.core.types.Expression<String> password, com.querydsl.core.types.Expression<String> name, com.querydsl.core.types.Expression<Integer> age) {
        super(CustomerDTO.class, new Class<?>[]{long.class, String.class, String.class, String.class, int.class}, id, loginId, password, name, age);
    }

}
```  

![ConstructorExpression](/assets/img/custom/querydsl-builder-projection/ConstructorExpression.jpg)  

`ConstructorExpression` 클래스를 상속하는 것을 확인할 수 있습니다. 해당 클래스의 생성자를 살펴봅시다. 생성자의 인자로 제네릭 클래스 타입과 프로젝션의 타입을 담은 배열, 그리고 프로젝션의 값을 인자로 받고 있는 것을 확인할 수 있습니다.  

![ConstructorExpressionConstructor](/assets/img/custom/querydsl-builder-projection/ConstructorExpressionConstructor.JPG)  

아래로 내래보면 `newInstance(Object... args)` 라는 메소드를 확인할 수 있습니다. 이 메소드가 바로 프로젝션을 생성하는 메소드 입니다.  

![newInstance](/assets/img/custom/querydsl-builder-projection/newInstance.JPG)  

생성자와 메소드를 통해 어떻게 생성자를 통해 조회 결과를 반환 받을수 있는지 확인할 수 있습니다.  

## **2. 새로운 Expression 클래스 만들기**  
위에서 살펴본 `ConstructorExpression` 은 어디까지나 클래스의 모든 필드를 매개변수로 갖는 생성자를 위한 조회 방식이기 때문에 빌더패턴을 사용할 수 없습니다. `FactoryExpressionBase` 클래스를 상속하는 새로운 프로젝션 클래스를 만드는게 낫겠네요.  

```java
 public KcExpression(Class<? extends T> type, Map<String, Expression<?>> bindings) {
    super(type);
    this.type = type;

    if (bindings == null || bindings.isEmpty()) {
        throw new RuntimeException("bindings must not be null or empty!");
    }

    this.bindings = Collections.unmodifiableMap(
            bindings
                    .entrySet()
                    .stream()
                    .filter(entry -> entry.getValue() != null)
                    .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue, (x, y) -> y, LinkedHashMap::new))
    );
}
```  

프로젝션 클래스의 생성자 입니다. 클래스 타입과 `Map<String, Expression<?>>`을 매개변수로 받습니다. 전달받은 map에서 null값을 제외하고 순서를 보장하는 `LinkedHashMap`으로 다시 생성하여 bindings 변수에 할당합니다.  

```java
@Override
public T newInstance(Object... a) {
    try {
        var arr = this.bindings.keySet().toArray();

        var rv = this.type.getDeclaredConstructor().newInstance();
        for (var i = 0; i < a.length; i ++) {
            var value = a[i];
            if (value != null) {
                var field = this.type.getDeclaredField((String) arr[i]);
                field.setAccessible(true);
                field.set(rv, value);
            }
        }

        return rv;
    } catch (InstantiationException | IllegalAccessException | NoSuchFieldException | InvocationTargetException | NoSuchMethodException e) {
        throw new ExpressionException(e.getMessage(), e);
    }
}
```  

`newInstance` 메소드에서는 리플렉션으로 빈 생성자를 생성하고 값이 넘어온 순서대로 필드에 값을 저정합니다. 문제는 이 메소에는 필드의 정보나 타입같은건 넘어오지 않는다는 점입니다.  
오직 `Object...` 형식으로 순수한 값만 넘어오기 때문에 현재 넘어오는 값이 어떠한 필드에 매핑되는지 알 수 없습니다. 그래도 다행인 것은 값이 필드 순서대로 넘어온다는 것입니다.  

제가 위에서 순서를 보장하는 `LinkedHashMap`을 사용했던 것은 위와같은 이유 때문입니다. 물론 애초에 이 클래스가 생성되는 시점에도 순서가 보장된 map을 파라미터로 넘겨야 합니다. 
메소드에 넘어오는 값이 순서를 보장하기 때문에 클래스의 필드정보를 담고 있는 map 객체의 요소가 반환받을 클래스의 필드 선언 순서와 동일하다면 문제없이 반환받을 클래스에 값을 저장하고 넘겨받을 수 있겠죠.  

## **3. record를 반환 받을 수 있는 Expression 만들기**  
위에서 만든 Expression 클래스에는 문제가 하나 있습니다. 빈 생성자를 만들고 그 이후에 값을 세팅하는 방식이기 때문에 빈 생성자가 존재할 수 없는 `record` 타입의 클래스는 사용할 수 없습니다.  

이를 위해 `record` 클래스 에 사용할 Expression 클래스는 따로 만들도록 하겠습니다.

```java
public class KcRecordExpression<T extends Record> extends KcExpression<T> {

    private final Class<? extends T> type;

    public KcRecordExpression(Class<? extends T> type, Map<String, Expression<?>> bindings) {
        super(type, bindings);
        if (!type.isRecord()) {
            throw new RuntimeException("This expression is an expression for record. Use KcExpression to bind a regular class.");
        }

        this.type = type;
    }

    @Override
    public T newInstance(Object... a) {
        if (this.type.getDeclaredFields().length != a.length) {
            throw new RuntimeException("Because a record type must create an object as a constructor that accepts all variables as arguments, the number of declared fields and the number of parameters to bind must be the same.");
        }

        try {
            var arr = getBindings().keySet().toArray();
            var fields = new Field[a.length];
            for (var i = 0; i < fields.length; i ++) {
                fields[i] = this.type.getDeclaredField((String) arr[i]);
            }

            var matchedConstructor = Arrays
                    .stream(this.type.getConstructors())
                    .filter(constructor -> {
                        var parameterTypes = constructor.getParameterTypes();
                        for (var i = 0; i < a.length; i ++) {
                            var constructorType = parameterTypes[i];
                            var field = fields[i];
                            if (!constructorType.getTypeName().equals(field.getType().getTypeName())) {
                                return false;
                            }
                        }

                        return true;
                    }).toList();

            if (matchedConstructor.size() != 1) {
                throw new RuntimeException("No or more than one constructor exists with the same number and type of parameters. It seems record is not suitable for this case.");
            }

            var constructor = matchedConstructor.get(0);
            return (T) constructor.newInstance(a);
        } catch (Exception e) {
            throw new ExpressionException(e.getMessage(), e);
        }
    }
}
```  

클래스 생성 시점에 타입이 `record` 타입인지 검사합니다.  

`newInstance` 메소드에는 방어 코드가 존재합니다. 첫째로 반환받을 `record` 클래스의 필드 갯수와 메소드에 넘어온 파라미터의 갯수가 똑같아야 합니다. 클래스 내에 있는 모든 변수들을 인수로 받을 생성자를 만들어야 하기 때문에 당연합니다.  

둘째로 사용할 생성자가 명확해야 합니다. `record` 클래스 내에도 생성자는 여러개 존재할 수 있습니다. 그러나 여기서 사용할 생성자는 자바가 기본으로 만들어주는 모든 변수를 인수로 갖는 생성자여야 합니다. 따라서 값의 타입과 매개변수의 타입, 생성자 매개변수의 숫자를 검증하여 생성자의 조건과 일치하는 단 하나의 생성자만을 사용하도록 하였습니다.  

## **4. AnnotationProcessor 만들기**  
커스텀 어노테이션을 만들고 해당 어노테이션이 붙어있는 클래스를 위에서 만든 프로젝션 클래스를 상속하는 클래스로써 새롭게 만들기 위해 AnnotationProcessor를 만들어야 합니다.  

첫째로 클래스 위에 붙일 어노테이션 입니다.  

```java
@Documented
@Target(ElementType.TYPE)
@Retention(RUNTIME)
public @interface KcQueryProjection {
}
```  

다음은 `AbstractProcessor`를 상속하는 AnnotationProcessor 클래스 입니다. 전체 코드는 이 글 하단의 링크에서 확인하실 수 있습니다.  

뭔가 특별한 방식이 있을것 같지만 전 그냥 다음과 같이 무식하게 작성하였습니다.  

![annotation-processor](/assets/img/custom/querydsl-builder-projection/annotation-processor.JPG)  

여기서 중요한 점은 세가지 입니다.
1. 현재 클래스가 `record` 인지 일반 클래스인지 구분 -> Expression 클래스가 다름
2. 제네릭 타입으로 원시 타입을 사용할 수 없기 때문에 원시 타입을 참조 타입으로 변경
3. 바인딩 map 객체를 만들때는 순서를 보장하는 `LinkedHashMap` 사용 (위에서 설명)  

이제 현재 프로젝트가 AnnotationProcessor로써 동작하게 하기 위해 추가로 필요한 작업을 진행합니다.  
1. resources/META-INF/services/javax.annotation.processing.Processor 파일에 AnnotationProcessor 클래스 명시  
2. pom.xml 에 아래 플러그인을 추가하여 컴파일러가 현재 프로젝트를 컴파일할때 annotation processing 작업을 건너 뛰도록 함

![proc-none](/assets/img/custom/querydsl-builder-projection/proc-none.JPG)  

필요한 작업을 진행하였다면 다음 명령어를 실행하여 local maven repository에 현재 프로젝트가 배포될 수 있도록 합니다.  

```
clean install -DskipTests
```

## **5. 테스트**
이제 새로운 프로젝트를 하나 만들고 위에서 만든 작업물을 테스트해 보도록 하겠습니다.  

배포된 프로젝트와 `querydsl-apt`를 의존성에 추가하고 `maven compiler plugin`을 추가하고 annotationProcessor를 등록합니다.  

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>

            <configuration>
                <source>17</source>
                <target>17</target>

                <generatedSourcesDirectory>target/generated-sources/querydsl</generatedSourcesDirectory>
                <annotationProcessors>
                    <annotationProcessor>
                        com.querydsl.apt.jpa.JPAAnnotationProcessor
                    </annotationProcessor>
                    <annotationProcessor>
                        com.keencho.querydsl.KcQuerydslAnnotationProcessor
                    </annotationProcessor>
                </annotationProcessors>
            </configuration>
        </plugin>
    </plugins>
</build>
```

이제 반환받기 원하는 class, record에 어노테이션을 붙이고 프로젝트를 컴파일 합니다.  

> :warning: 프로젝션 클래스에서 빈 생성자를 필요로 하기 때문에 아래 어노테이션이 붙은 클래스에는 빈 생성자가 필수로 존재해야 합니다. (없다면 NoSuchMethodException 발생) 

```java
@KcQueryProjection
public record CustomerRecord(
        Long id,
        String loginId,
        String password,
        String name
) { }
```  

테스트 코드를 작성하고 테스트를 수행합니다.

```java
var q = QCustomer.customer;

var predicate = new BooleanBuilder();
predicate.and(q.name.contains("1"));

var bindings = KcQCustomerDTO.builder()
        .id(q.id)
        .name(q.name)
        .loginId(q.loginId)
        .password(q.password)
        .build();

var list = jpaQueryFactory
        .select(bindings)
        .from(q)
        .where(predicate)
        .orderBy(q.loginId.asc())
        .fetch();

var bindings2 = KcQCustomerRecord.builder()
        .id(q.id)
        .name(q.name)
        .loginId(q.loginId)
        .password(q.password)
        .build();

var list2 = jpaQueryFactory
        .select(bindings2)
        .from(q)
        .where(predicate)
        .orderBy(q.loginId.asc())
        .fetch();

for (var i = 0; i < list.size(); i ++) {
    Assert.isTrue(list.get(i).getName().equals(list2.get(i).name()));
    Assert.isTrue(list.get(i).getPassword().equals(list2.get(i).password()));
    Assert.isTrue(list.get(i).getId().equals(list2.get(i).id()));
    Assert.isTrue(list.get(i).getLoginId().equals(list2.get(i).loginId()));
}
```  

# **코드 확인** 
전체 코드는 [이곳](https://github.com/keencho/java-sandbox/tree/master/blog-example-code/querydsl-builder-projection) 에서 확인하실 수 있습니다.

