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
> java17, gradle, querydsl-*:5.0.0 을 사용합니다.

## **1. gradle plugin 분석**
gradle로 환경을 구성하는 분들이시라면 아마 ewerk-plugin을 사용해 querydsl을 컴파일하실 것입니다. build.gradle에는 다음과 같이 정의되어 있겠죠.

```gradle
plugins {
    ...
    id 'com.ewerk.gradle.plugins.querydsl' version '1.0.10'
}

...

// querydsl
def querydslDir = "$buildDir/generated/querydsl"

querydsl {
    jpa = true
    querydslSourcesDir = querydslDir
}

sourceSets {
    main.java.srcDir querydslDir
}

configurations {
    compileOnly {
        extendsFrom annotationProcessor
    }

    querydsl.extendsFrom compileClasspath
}

compileQuerydsl {
    options.annotationProcessorPath = configurations.querydsl
}
```

@QueryProjection 어노테이션이 어떻게 코드를 생성하는지 확인해봐야 하기 때문에 이 플러그인을 분석해보겠습니다. [여기](https://github.com/ewerk/gradle-plugins/tree/master/querydsl-plugin) 에서 코드를 확인해 보실수 있습니다.

쭉 코드를 분석해보면 [이 클래스](https://github.com/ewerk/gradle-plugins/blob/master/querydsl-plugin/src/main/groovy/com/ewerk/gradle/plugins/QuerydslPluginExtension.groovy) 에서 annotation processor를 지정하는것을 확인할 수 있습니다.

gradle 파일의 querydsl 블럭을 보면 jpa가 true로 세팅된것을 확인할수 있고, 이에따라 `com.querydsl.apt.jpa.JPAAnnotationProcessor` 를 사용한다는 것을 확인할 수 있습니다. IDE에서 해당 클래스를 찾아봅시다.

```java
package com.querydsl.apt.jpa;

@SupportedAnnotationTypes({"com.querydsl.core.annotations.*", "jakarta.persistence.*", "javax.persistence.*"})
public class JPAAnnotationProcessor extends AbstractQuerydslProcessor {

    @Override
    protected Configuration createConfiguration(RoundEnvironment roundEnv) {
        Class<? extends Annotation> entity = Entity.class;
        Class<? extends Annotation> superType = MappedSuperclass.class;
        Class<? extends Annotation> embeddable = Embeddable.class;
        Class<? extends Annotation> embedded = Embedded.class;
        Class<? extends Annotation> skip = Transient.class;
        return new JPAConfiguration(roundEnv, processingEnv,
                entity, superType, embeddable, embedded, skip);
    }

}
```

부모클래스인 `AbstractQuerydslProcessor` 를 확인해보면 `process(...)` 메소드에 의해 Q클래스들이 생성되는것을 확인할 수 있습니다.

```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    setLogInfo();
    logInfo("Running " + getClass().getSimpleName());

    if (roundEnv.processingOver() || annotations.size() == 0) {
        return ALLOW_OTHER_PROCESSORS_TO_CLAIM_ANNOTATIONS;
    }
.....
```

그렇다면 `JPAAnnotationProcessor` 를 상속받는 annotation processor를 만들고 그 안에서 파일을 생성하는 코드를 작성하면 되겠네요.

## **2. 어노테이션 및 QBean 상속 클래스 만들기**
먼저 어노테이션을 만들겠습니다. 이 어노테이션은 클래스에 붙을수 있는 어노테이션이며 이 어노테이션이 붙은 클래스는 파일 생성 대상으로 지정됩니다. 참고로 **이 어노테이션이 붙은 클래스에는 기본 빈 생성자가 꼭 존재해야 합니다.**
```java
@Documented
@Target(ElementType.TYPE)
@Retention(RUNTIME)
public @interface KcQueryProjection {
}
```

다음은 QBean을 상속하는 클래스를 만들겠습니다. `Projections.fields(...)` 와 `Projections.bean(...)` 메소드가 QBean 클래스를 리턴하기 때문에 이를 선택했습니다.

```java
public class KcQBean<T> extends QBean<T> {

    private final Class<? extends T> type;

    public KcQBean(Class<? extends T> type) {
        super(type);
        this.type = type;
    }

    public KcQBean(Class<? extends T> type, Map<String, Expression<?>> bindings) {
        super(type, true, bindings);
        this.type = type;
    }

    public QBean<T> build() {
        Map<String, Expression<?>> bindings = new HashMap<>();

        for (var declaredField : this.getClass().getDeclaredFields()) {
            var modifiers = declaredField.getModifiers();
            if (!Modifier.isFinal(modifiers) && !Modifier.isStatic(modifiers)) {
                declaredField.setAccessible(true);
                try {
                    var v = declaredField.get(this);
                    if (v != null) {
                        bindings.put(declaredField.getName(), (Expression<?>) v);
                    }
                } catch (IllegalAccessException e) {
                    throw new KcSystemException(e.getMessage());
                }
            }
        }

        return new KcQBean<>(this.type, bindings);
    }
}
```

첫번째 생성자는 조회할 DTO를 만들때 사용됩니다. 두번째 생성자는 실제 쿼리가 날라가기 이전에 build() 메소드에 의해 사용됩니다. 결국 객체를 두번생성하는 것이기 때문에 메모리 낭비라고 볼수도 있겠네요.

사실 QBean 클래스의 생성자의 접근제어자가 public 이었다면 객체를 한번만 생성해도 되지만 아쉽게도 생성자의 접근제어자가 모두 protected이기 때문에 어쩔수 없는 선택이었습니다.

## **3. 코드 Writer, Serializer**
querydsl-apt 이 Q 코드를 생성할때는 `com.querydsl.codegen.Serializer` 인터페이스를 상속받은 serializer들과 `com.querydsl.codegen.utils.JavaWriter`를 사용합니다. 예를들어 @QueryProjection 대상 클래스를 생성할 경우 `com.querydsl.codegen.DefaultProjectionSerializer` 를 사용합니다.

어쨌든 이 기능들이 모두 필요하지는 않습니다. 딱 필요한 메소드들만 담아 클래스를 두개 생성합니다.

```java
public class KcJavaWriter {

    private static final String PRIVATE = "private ";

    private static final String PUBLIC = "public ";

    private final Set<String> classes = new HashSet<>();

    private final Set<String> packages = new HashSet<>();

    private static final String BUILDER = "builder ";

    private final Appendable appendable;
    private String indent = "";
    private final int spaces;
    private final String spacesString;

    public KcJavaWriter(Appendable appendable) {
        this.appendable = appendable;
        this.spaces = 4;
        this.spacesString = StringUtils.repeat(' ', spaces);
        this.packages.add("java.lang");
    }

    private KcJavaWriter param(Parameter parameter) throws IOException {
        append(parameter.getType().getGenericName(true, packages, classes));
        append(" ");
        append(parameter.getName());
        return this;
    }

    public KcJavaWriter nl() throws IOException {
        return append(System.lineSeparator());
    }

    public KcJavaWriter line(String... segments) throws IOException {
        append(indent);
        for (String segment : segments) {
            append(segment);
        }
        return nl();
    }

    public KcJavaWriter append(CharSequence csq) throws IOException {
        appendable.append(csq);
        return this;
    }

    public KcJavaWriter beginLine(String... segments) throws IOException {
        append(indent);
        for (String segment : segments) {
            append(segment);
        }
        return this;
    }

    public KcJavaWriter goIn() {
        indent += spacesString;
        return this;
    }

    public KcJavaWriter goOut() {
        if (indent.length() >= spaces) {
            indent = indent.substring(0, indent.length() - spaces);
        }
        return this;
    }

    public KcJavaWriter privateExpressionField(Type type, String name) throws IOException {
        return beginLine(PRIVATE)
                .param(new Parameter(name, new ClassType(Expression.class, type)))
                .append(Symbols.SEMICOLON)
                .nl().nl();
    }

    public KcJavaWriter setterMethod(Type type, String name) throws IOException {
        return beginLine(PUBLIC)
                .param(new Parameter("set" + StringUtils.capitalize(name), new ClassType(void.class)))
                .append("(")
                .param(new Parameter(name, new ClassType(Expression.class, type)))
                .append(") {")
                .nl()
                .goIn()
                .beginLine(String.format("this.%s = %s;", name, name))
                .nl()
                .goOut()
                .beginLine("}")
                .nl().nl();
    }

    public KcJavaWriter builderMethod(Type type, String name) throws IOException {
        return beginLine(PUBLIC).append(StringUtils.capitalize(BUILDER))
                .append(name).append("(").param(new Parameter(name, new ClassType(Expression.class, type))).append(") {")
                .nl()
                .goIn()
                .beginLine(String.format("this.%s = %s;", name, name))
                .nl()
                .beginLine("return this;")
                .nl()
                .goOut()
                .beginLine("}")
                .nl().nl();
    }
}
```

```java
public class KcProjectionSerializer {

    private static final String CLASS_PREFIX = "KcQ";

    public static void serialize(final EntityType model, @NonNull KcJavaWriter writer) throws IOException {

        // package
        writer.line("package ", getPackageWithoutClassName(model), ";").nl();

        // imports
        writer.line("import ", NumberExpression.class.getPackageName(), ".*;");
        writer.line("import ", KcQBean.class.getName(), ";");

        // javadoc
        writer.line("/**");
        writer.line(" * " + getKcFullPackageName(model) + " is a KcQuerydsl Projection type for " + model.getSimpleName());
        writer.line(" */");

        // generated annotation
        writer.line("@", Generated.class.getName(), "(\"", KcProjectionSerializer.class.getName(), "\")");

        // init class
        String className = getKcQClassName(model);

        writer.beginLine("public class " + className);
        writer.append(" extends ").append(KcQBean.class.getSimpleName()).append("<").append(model.getSimpleName()).append(">").append(" {");
        writer.nl().nl();

        // empty constructor
        writer.goIn();
        writer.line("public ", className, "() {");
        writer.goIn();
        writer.line("super(", model.getSimpleName(), ".class);");
        writer.goOut();
        writer.line("}");
        writer.nl();

        // builder constructor
        writer.line("public ", className, "(Builder builder) {");
        writer.goIn();
        writer.line("super(", model.getSimpleName(), ".class);");

        for (var property : model.getProperties()) {
            var name = property.getName();
            writer.line("this.", name, " = builder.", name, ";");
        }

        writer.goOut();
        writer.line("}");
        writer.nl();

        // serialVersionUID
        writer.line("private static final long serialVersionUID = ", model.hashCode() + "L;");
        writer.nl();

        // field / setter
        for (var property : model.getProperties()) {
            var type = property.getType();
            var name = property.getName();

            writer.privateExpressionField(type, name);
            writer.setterMethod(type, name);
        }

        // builder method
        writer.line("public static Builder builder() {");
        writer.goIn();
        writer.line("return new Builder();");
        writer.goOut();
        writer.line("}");
        writer.nl();

        // builder class
        writer.line("public static class Builder {");
        writer.nl();
        writer.goIn();

        // builder class field / build method
        for (var property : model.getProperties()) {
            var type = property.getType();
            var name = property.getName();

            writer.privateExpressionField(type, name);
            writer.builderMethod(type, name);
        }

        // builder class final build method
        writer.line("public ", className, " build() {");
        writer.goIn();
        writer.line("return new ", className, "(this);");
        writer.goOut();
        writer.line("}");
        writer.nl();

        // close builder class
        writer.goOut();
        writer.line("}");
        writer.nl();

        // close class
        writer.goOut();
        writer.line("}");
    }

    public static String getPackageWithoutClassName(EntityType entityType) {
        return entityType.getInnerType().getPackageName();
    }

    public static String getKcFullPackageName(EntityType entityType) {
        return getPackageWithoutClassName(entityType) + "." + CLASS_PREFIX + entityType.getInnerType().getSimpleName();
    }

    public static String getKcQClassName(EntityType entityType) {
        return CLASS_PREFIX + entityType.getInnerType().getSimpleName();
    }

}
```

위 코드를 보시면 아시겠지만 package 정의, import, class... 모두 하나하나 파일에 입력하는 방식을 사용하고 있습니다. 항상 궁금해 했던 `Q 객체는 어떻게 생성되는 것일까?` 에 대한 궁금증도 해결되었네요.

만들어질 클래스는 위에서 만든 `KcQBean` 클래스를 상속하게 되어있습니다. 또한 builder 메소드와 setter 메소드를 생성하여 생성자 방식 에서 벗어나게 되었습니다. 이펙티브 자바에 의하면 매개변수가 많은 경우 setter 대신 builder를 사용하라고 하였기 떄문에 builder / setter 둘다 여는것 보다는 builder는 기본으로 지정하고 setter는 어노테이션의 옵션으로 생성여부를 받는것도 좋은 방법이겠네요.

또하나 보시면 클래스앞에 `Kc` 라는 prefix를 붙여 생성합니다. 이는 `@QueryProjection`과의 공존을 위한 것으로 `Q~` 형태의 클래스를 사용하려면 간단히 이미 만들어진 QClass 들을 override 하면 되지만 기존 `Q~` 클래스는 사라지게 된다는 단점이 있습니다. 저는 둘다 사용하기 위해 `Kc` 라는 prefix를 사용하였습니다.

## **4. annotation processor**
마지막 annotation processor 코드입니다. ewerk plugin에 의해 실행될 클래스를 정의합니다.

```java
@SupportedAnnotationTypes({"com.querydsl.core.annotations.*", "jakarta.persistence.*", "javax.persistence.*"})
public class KcQuerydslAnnotationProcessor extends JPAAnnotationProcessor {

    private RoundEnvironment roundEnv;
    private Configuration conf;
    private ExtendedTypeFactory typeFactory;

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        var result = super.process(annotations, roundEnv);

        this.roundEnv = roundEnv;
        this.conf = super.createConfiguration(this.roundEnv);
        this.typeFactory = new ExtendedTypeFactory(processingEnv, conf.getEntityAnnotations(), conf.getTypeMappings(), conf.getQueryTypeFactory(), conf.getVariableNameFunction());
        this.generateAndSerialize();

        return result;
    }

    private void generateAndSerialize() {
        var kcQueryProjectionElement = this.roundEnv.getElementsAnnotatedWith(KcQueryProjection.class);

        for (var element : kcQueryProjectionElement) {
            var typeElement = (TypeElement) element;
            var model = this.getEntityType(typeElement);

            var serializer = new KcProjectionSerializer();
            var fullPackageClassName = serializer.getKcFullPackageName(model);

            try (Writer w = conf.getFiler().createFile(processingEnv, fullPackageClassName, Collections.singleton(typeElement))) {
                var writer = new KcJavaWriter(w);
                serializer.serialize(model, writer);
            } catch (IOException ignored) { }

        }
    }

    private EntityType getEntityType(TypeElement typeElement) {
        var type = this.typeFactory.getType(typeElement.asType(), true);
        var entityType = new EntityType(type.as(TypeCategory.ENTITY), this.conf.getVariableNameFunction());

        for (var field : ElementFilter.fieldsIn(typeElement.getEnclosedElements())) {
            entityType.addProperty(new Property(entityType, field.getSimpleName().toString(), this.typeFactory.getType(field.asType(), true)));
        }

        return entityType;
    }

}
```

기존의 작업을 `super.process(...)` 로 수행한 후에 `KcQueryProjection` 어노테이션이 붙어있는 클래스들을 대상으로 지정하고 코드를 생성합니다.

주의할점이 하나 있습니다. 저는 위에서 설명했다시피 `Kc` 라는 prefix를 붙여 클래스를 생성합니다. 만약 prefix를 붙이지 않고 클래스를 생성한다면 `serialize()` 메소드의
```java
try (Writer w = conf.getFiler().createFile(processingEnv, fullPackageClassName, this.typeElements.get(val.getFullName()))) {
    var writer = new KcJavaWriter(w);
    KcProjectionSerializer.serialize(val, writer);
} catch (IOException e) {
    throw new KcRuntimeException(e.getMessage());
}
```
부분에서 파일을 재생성 할 수 없다는 에러가 발생하게 됩니다. 만약 기존 q 파일을 override 하기 원하시는 분들이라면 위 코드는 다른 코드로 대체되어야 합니다.

## **5. plugin 관련 설정**
필요한 코드는 모두 작성하였습니다. 위에서 만든 `KcQuerydslAnnotationProcessor` 가 수행되게 만들어야 하는데요, 아까 잠시 살펴본 ewerk-plugin을 다시 살펴봅시다.

`QuerydslPlugin.groovy` 클래스를 보면 아래와 같은 메소드가 있습니다.
```groovy
private static void applyCompilerOptions(Project project) {
    project.tasks.compileQuerydsl.options.compilerArgs += [
        "-proc:only",
        "-processor", project.querydsl.processors()
    ]

    if(project.querydsl.aptOptions.size() > 0){
        for(aptOption in project.querydsl.aptOptions) {
            project.tasks.compileQuerydsl.options.compilerArgs << "-A" + aptOption
        }
    }
}
```

컴파일러 옵션을 지정하는 시점에 `-processor` 로 프로세서를 지정하는 코드가 있네요. 그렇다면 이부분을 건드려 위에서 만든 processor가 수행되게 하면 되겠네요!

build.gradle의 compileQuerydsl 블록을 다음과 같이 수정합니다.

```groovy
compileQuerydsl {
    options.annotationProcessorPath = configurations.querydsl

    doFirst {
        options.compilerArgs = ['-proc:only', '-processor', 'com.keencho.lib.spring.jpa.querydsl.KcQuerydslAnnotationProcessor']
    }
}
```

querydsl을 컴파일하기 전에 기존의 코드를 무시하고 `KcQuerydslAnnotationProcessor` 가 수행될수 있게 하였습니다.

이 글의 서두에서 주의했듯이 위에서 살펴본 자바 코드가 작성된 프로젝트와 build.gradle이 속해있는 프로젝트는 달라야 합니다. 프로젝트가 같다면 A를 컴파일하기위해 A가 사용되어야 하기 때문에 작업을 수행할수 없겠죠.

이제 기존처럼 `compileQuerydsl` task를 수행하면 끝입니다! 간단한 예제 코드와 함께 잘 동작하는지 확인해 보도록 하겠습니다.

# **테스트**
모델과 DTO 클래스는 따로 작성하지 않겠습니다. `이런식으로 조회할수 있다` 정도만 봐주시면 좋을것 같습니다. `compileQuerydsl` task 수행후 아래와 같은 테스트 코드를 작성하였습니다.

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

    var bb = new BooleanBuilder();
    bb.and(q.fromAddress.startsWith("c"));

    var list = queryFactory
            .select(simpleDTO.build())
            .from(q)
            .where(predicate)
            .fetch();

    System.out.println(list.size());
}
```

정상적으로 조회되었고 다음과 같은 쿼리가 수행되었습니다.
```sql
select
    delivery0_.delivery_id as col_0_0_,
    delivery0_.from_address as col_1_0_,
    delivery0_.order_order_id as col_2_0_,
    delivery0_.from_number as col_3_0_,
    delivery0_.from_name as col_4_0_,
    delivery0_.from_address as col_5_0_
from
    delivery delivery0_
where
    delivery0_.from_address like 'c%' escape '!'
```

객체안에 `deliveryDTO`객체를 넣어도 알아서 잘 불러오는것까지 확인하였습니다. `Map<String, Expressions<?>>` 형태의 맵에 질의할 데이터가 다 들어가기 때문에 잘 불러와지는것 같네요.

제 개인적으로는 조회할 객체 안에 또다른 객체를 넣기보다는 case문이나 기타 sql문을 최대한 활용하여 조회할 객체 안에는 필드만 존재하게 하는것이 바람직하다고 생각합니다.

# **마치며**
개인적으로 이 라이브러리를 만들며 querydsl의 동작방식, q클래스의 생성원리등 querydsl을 사용하며 궁금했던 부분을 시원하게 해결하였습니다.

**querydsl에 지나치게 의존적이다**라는 단점이 있지만 orm을 사용한 이상 코드에 의존하게 되는게 단점인지는 모르겠네요. 컴파일시점에 에러를 잡을 확률가게 된다는 장점도 존재하니까요. 실무에서 써봐야 알겠지만 기존방식에 비해 생산성이 향상되었으면 좋겠습니다.


> 전체 코드: [https://github.com/keencho/lib-spring/tree/master/src/main/java/com/keencho/lib/spring/jpa/querydsl](https://github.com/keencho/lib-spring/tree/master/src/main/java/com/keencho/lib/spring/jpa/querydsl)



