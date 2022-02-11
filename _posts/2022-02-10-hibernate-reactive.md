---
title: Hibernate Reactive 
author: keencho
date: 2022-02-10 10:12:00 +0900
categories: [Java]
tags: [Java, Hibernate, Webflux]
---

# **개요**
[Hibernate Reactive](https://hibernate.org/reactive/)가 드디어 정식 출시 되었습니다. (~~사실 현재기준 3달 지났습니다.~~)  

`Webflux - R2DBC`로 프로젝트를 진행할때 ORM에 대한 갈망을 느끼곤 했었는데, 확실히 갈증을 해소해줄 만한 물건이 나온것 같습니다.

현재는 `Quarkus`, `Panache`와 사용했을때 더 효율성을 발휘하는 것으로 보입니다. 그래도 예전 `Spring Data JPA`가 나오기전 Hibernate를 사용할때를 떠올리며 내용을 정리해 보았습니다. 언젠가는 `Spring Data Family`에 포함되어 `Spring Data Reactive JPA(?)`가 나오길 희망해 봅니다.  

## **Hibernate Reactive**  
Webflux가 나온지 4년이 지났습니다. `Spring Data R2DBC` 를 통해 non-blocking 하게 데이터베이스에 접근할 수 있지만, 이렇다할 ORM 구현체는 없는 상황이었습니다.  

Hibernate Reactive는 Hibernate ORM 을 위한 리액티브 API로, 데이터베이스에 non-blocking하게 접근할 수 있도록 지원합니다. Hibernate Reactive는 데이터베이스와 non-blocking 방식으로 통신하도록 설계된 최초의 ORM 구현체 입니다.  

또한 많은 부분이 기존의 Hibernate ORM, JPA 2.2 과 동일하기 때문에, 만약 JPA에 대한 이해도가 깊은 분이라면 쉽게 Hibernate Reactive에 적응하실 것이라 생각합니다.

## **소개 및 예제**  
이어질 내용 부터는 간단한 CRUD 어플리케이션을 만들면서 Hibernate Reactive를 소개해 보겠습니다.  

## **1. 프로젝트 세팅**  
Spring 진영에서 아직까지는 Hibernate Reactive를 받아들이지 않은 것으로 보입니다만, 다행히도 Spring과 Hibernate Reactive를 조합하는 것은 그리 어렵지 않은것 같습니다. 따라서 저는 `Spring Webflux` 기반의 프로젝트를 만들어 보겠습니다.  

### **1.1. 라이브러리 추가**  
사용할 라이브러리들을 정의한 gradle 파일 입니다.  

```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webflux'
    implementation 'org.hibernate.reactive:hibernate-reactive-core:1.1.2.Final'
    implementation 'io.vertx:vertx-pg-client:4.2.4'
    implementation 'com.ongres.scram:client:2.1'
    implementation 'io.smallrye.reactive:mutiny-reactor:1.3.1'
    implementation 'org.projectlombok:lombok'
}
```  

> 나중에 com.ongres.scram 클래스가 없다는 에러가 발생시 패키지에 `implementation 'org.hibernate.orm:hibernate-jpamodelgen:6.0.0.CR1'` 를 추가해보세요.  

이 프로젝트는 PostgreSQL 을 기반으로 진행될 예정 이지만 현재 사용할 수 있는 데이터베이스와 드라이버는 다음과 같습니다.

|Database|Driver dependency|
|-----|-----|  
|PostgreSQL or CockroachDB|io.vertx:vertx-pg-client:{vertxVersion}|
|MySQL or MariaDB|io.vertx:vertx-mysql-client:{vertxVersion}|
|DB2|io.vertx:vertx-db2-client:{vertxVersion}|
|SQL Server|io.vertx:vertx-mssql-client:${vertxVersion}|  

### **1.2. persistence.xml 정의**
Hibernate Reactive는 표준 JPA persistence.xml 문서를 통해 구성되며, 보통 /META-INF 디렉토리에 배치되어야 합니다.  

```xml
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd"
             version="2.2">
    <persistence-unit name="pu">
        <provider>org.hibernate.reactive.provider.ReactivePersistenceProvider</provider>

        <properties>

            <!-- PostgreSQL -->
            <property name="javax.persistence.jdbc.url"
                      value="jdbc:postgresql://localhost:5432/database"/>

            <!-- Credentials -->
            <property name="javax.persistence.jdbc.user"
                      value="user"/>
            <property name="javax.persistence.jdbc.password"
                      value="password"/>

            <!-- The Vert.x SQL Client connection pool size -->
            <property name="hibernate.connection.pool_size"
                      value="10"/>

            <!-- Automatic schema export -->
            <property name="javax.persistence.schema-generation.database.action"
                      value="update"/>

            <!-- SQL statement logging -->
            <property name="hibernate.show_sql" value="true"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.highlight_sql" value="true"/>
        </properties>
    </persistence-unit>
</persistence>
```  

- Hibernate Reactive에 유일한 필수 조건은 <provier> 요소이며, 이는 꼭 명시되어야 합니다.  
- 구성 속성중 JDBC라고 명시되어 있지만 Hibernate Reactive는 JDBC가 없으며 과거 JPA 사양에서 정의한 레거시 속성 이름을 뿐입니다. 특히 Hibernate Reactive는 자체적으로 JDBC URL을 읽고 해석합니다.  
- `hibernate.dialect` 를 지정할 필요는 없습니다. Hibernate Reactive는 자동으로 알맞은 dialect를 결정합니다.  
- Hibernate는 구성 가능한 많은 것들을 가지고 있지만 이것은 레거시 코드와의 호환성을 위해 유지되고 있을 뿐 JDBC와 JTA와의 직접적인 관련은 없습니다.  

다음은 자동 스키마 구성에 대한 내용입니다.

|프로퍼티명|설명|
|-----|-----|  
|javax.persistence.schema-generation.database.action|create: 스키마를 삭제하고 테이블, 시퀀스, 제약조건을 생성|
| |create-only: 테이블, 시퀀스, 제약조건을 생성|
| |create-drop: 스키마를 삭제하고 `SessionFactory`가 생성되는 시점에 재생성, 추가적으로 `SessionFactory`가 소멸되면 스키마를 삭제|
| |drop: `SessionFactory`가 소멸되는 시점에 스키마를 삭제|
| |validate: 변경없이 데이터베이스 스키마와 엔티티가 일치하는지 검증|
| |update: 엔티티와 스키마를 비교하여 스키마에 존재하지 않는 엔티티(혹은 필드)가 존재 한다면 스키마를 업데이트|
|javax.persistence.create-database-schemas|(선택사항) 만약 true 라면, 스키마와 카탈로그를 자동 생성|
|javax.persistence.schema-generation.create-source|(선택사항) 값이`metadata-then-script`혹은 `script-then-metadata` 테이블, 시퀀스를 생성할 때 추가 SQL 스크립트를 실행|
|javax.persistence.schema-generation.create-script-source|(선택사항) 바로 위에서 실행할 SQL 스크립트 이름|  

> 주의! Db2의 경우 `validate` 와 `update`를 지원하지 않습니다.  

### **1.3 SQL 로깅**  

|프로퍼티명|설명|
|-----|-----| 
|hibernate.format.sql|true라면, 멀티라인, 들여쓰기 형식으로 로그 기록|
|hibernate.highlight_sql|true라면, ANSI 이스케이프 코드를 통해 구문을 강조하는 형태로 로그 기록|  

위 두 옵션이 모두 true라면 아래와 같이 보이게 됩니다.  
![logging](/assets/img/custom/hibernate-reactive/logging.png)  

## **2. Java code 작성**  
본격적으로 자바 코드를 작성해 보겠습니다.  

```java
@Entity
@Table(name = "post")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    Long id;

    private String title;

    private String contents;

    @ManyToOne(targetEntity = Author.class)
    private Author author;
}
```  

```java
@Entity
@Table(name = "author")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Author {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    Long id;

    private String name;
}
```

작성자와 글을 N:1의 연관관계로 표현하였습니다.  

```java
@Bean
public Mutiny.SessionFactory sessionFactory() {
    return Persistence
            .createEntityManagerFactory("pu")
            .unwrap(Mutiny.SessionFactory.class);
}
```

`SessionFactory` 객체를 bean으로 등록합니다. 이때 `createEntityManagerFactory`에 들어갈 값은 `persistence.xml` 파일의 `persistence-unit name` 의 값과 동일해야 합니다. 

`Mutiny` 타입과 `Stage` 타입을 선택할 수 있는데요, 큰 차이는 없고 공식문서의 코드는 `Mutiny`를 선택해였기 때문에 여기서도 `Mutiny`를 사용하도록 하겠습니다.

```java
public interface KcReactiveRepository {

    <E> Uni<E> save(E entity);

    <E> Uni<List<E>> listAll(Class<E> clazz);

    <E> Uni<E> findById(Class<E> clazz, Object id);

    <E> Uni<Integer> deleteById(Class<E> clazz, Object id);

    <E> Uni<Integer> deleteAll(Class<E> clazz);
}
```

```java
@Component
public record KcReactiveRepositoryImpl(Mutiny.SessionFactory sessionFactory) implements KcReactiveRepository {

    @Override
    public <E> Uni<E> save(E entity) {
        Assert.notNull(entity, "entity must not be null!");

        return sessionFactory.withSession(session -> session.persist(entity).chain(session::flush).replaceWith(entity));
    }

    @Override
    public <E> Uni<List<E>> listAll(Class<E> clazz) {
        var query = this.buildQuery(clazz);

        query.from(clazz);

        return sessionFactory.withSession(session -> session.createQuery(query).getResultList());
    }

    @Override
    public <E> Uni<E> findById(Class<E> clazz, Object id) {
        Assert.notNull(id, "id must not be null!");

        return sessionFactory.withSession(session -> session.find(clazz, id));
    }

    @Override
    public <E> Uni<Integer> deleteById(Class<E> clazz, Object id) {
        Assert.notNull(id, "id must not be null!");

        var criteriaBuilder = this.sessionFactory.getCriteriaBuilder();

        CriteriaDelete<E> delete = criteriaBuilder.createCriteriaDelete(clazz);
        var root = delete.from(clazz);
        var idField = Arrays.stream(clazz.getDeclaredFields())
                .filter(field -> field.getAnnotation(Id.class) != null)
                .findFirst().orElse(null);

        Assert.notNull(idField, "id field of entity must not be null!");

        delete.where(criteriaBuilder.equal(root.get(idField.getName()), id));

        return sessionFactory.withTransaction((session, tx) -> session.createQuery(delete).executeUpdate());

    }

    @Override
    public <E> Uni<Integer> deleteAll(Class<E> clazz) {
        Assert.notNull(clazz, "class not not be null!");

        var criteriaBuilder = this.sessionFactory.getCriteriaBuilder();

        CriteriaDelete<E> delete = criteriaBuilder.createCriteriaDelete(clazz);
        delete.from(clazz);

        return sessionFactory.withTransaction((session, tx) -> session.createQuery(delete).executeUpdate());
    }

    private <E> CriteriaQuery<E> buildQuery(Class<E> clazz) {
        return sessionFactory.getCriteriaBuilder().createQuery(clazz);
    }
}
```  

대망의 리포지토리 코드입니다. 문서를 보며 만들다 보니 모듈화 할수있는 부분이 보여 모듈화하여 코드를 작성해 보았습니다. 간단한 CRUD 기능을 수행할 수 있는 리포지토리 입니다.  

중간에 `deleteById`의 경우 기본키가 하나일 경우만을 생각하고 작성하였습니다. 복합키는 고려대상이 아닙니다.  

`SessionFactory`를 의존성으로 주입받아야 하기 때문에 아래 bean을 등록해 줍니다.

```java
@Bean
public KcReactiveRepository kcReactiveRepository() {
    return new KcReactiveRepositoryImpl(
            Persistence
                    .createEntityManagerFactory("pu")
                    .unwrap(Mutiny.SessionFactory.class)
    );
}
```  

마지막으로, 위에서 작성한 리포지토리를 사용해 각각의 도메인에서 CRUD 기능을 수행하는 클래스들을 생성하였습니다.  

```java
@Repository
public class PostRepository {

    @Autowired
    KcReactiveRepository kcReactiveRepository;

    public Uni<Post> save(String title, String contents, Author author) {
        Post entity = Post.builder()
                .title(title)
                .contents(contents)
                .author(author)
                .build();

        return kcReactiveRepository.save(entity);
    }

    public Uni<List<Post>> listAll() {
        return kcReactiveRepository.listAll(Post.class);
    }

    public Uni<Post> findById(Long id) {
        return kcReactiveRepository.findById(Post.class, id);
    }

    public Uni<Integer> deleteAll() {
        return kcReactiveRepository.deleteAll(Post.class);
    }

    public Uni<Integer> deleteById(Long id) {
        return kcReactiveRepository.deleteById(Post.class, id);
    }

}
```  

```java
@Repository
public class AuthorRepository {

    @Autowired
    KcReactiveRepository kcReactiveRepository;

    public Uni<Author> save(String name) {
        Author entity = Author.builder()
                .name(name)
                .build();

        return kcReactiveRepository.save(entity);
    }

    public Uni<List<Author>> listAll() {
        return kcReactiveRepository.listAll(Author.class);
    }

    public Uni<Author> findById(Long id) {
        return kcReactiveRepository.findById(Author.class, id);
    }

    public Uni<Integer> deleteAll() {
        return kcReactiveRepository.deleteAll(Author.class);
    }

    public Uni<Integer> deleteById(Long id) {
        return kcReactiveRepository.deleteById(Author.class, id);
    }

}
```  

## **3. 테스트**  
코드 작성이 모두 끝났습니다. 기능이 잘 동작하는지 테스트를 수행해 보겠습니다.  

```java
@Test
void fullTest() {

    var simpleAuthor = authorRepository
            .save("홍길동")
            .await()
            .indefinitely();

    IntStream.range(1, 11).forEach(idx -> {
       var uni = postRepository.save("title" + idx, "contents"+ idx, simpleAuthor);
       uni.await().indefinitely();
    });

    ////////////////////////////////////////////////////////////////////////////////////////////////

    var authorList = authorRepository.listAll().await().indefinitely();
    var postList = postRepository.listAll().await().indefinitely();

    Assert.isTrue(authorList.size() == 1, "size of author list must be 1");
    Assert.isTrue(postList.size() == 10, "size of post list must be 10");

    long authorId = authorList.stream().findFirst().get().getId();
    long postId = postList.stream().findAny().get().getId();

    var deleteNotExistAuthor = authorRepository.deleteById(authorId + 1).await().indefinitely();
    var deleteExistPost = postRepository.deleteById(postId).await().indefinitely();

    Assert.isTrue(deleteNotExistAuthor == 0, "query result of delete empty author must be 0");
    Assert.isTrue(deleteExistPost == 1, "query result of delete post must be 1");

}
```  

1. author과 post를 insert 합니다.
2. 모든 author 과 post를 불러와 잘 insert 되었는지 확인합니다.
3. 삭제를 통해 쿼리 수행 결과가 잘 넘어오는지 확인합니다.  

만약 webflux 에서 Mono, Flux 객체를 리턴해야 한다면

```java
return ServerResponse.ok().body(postRepository.listAll().convert().with(toMono()));
```  

위와같이 `mutiny-reactor` 패키지에 있는 `toMono()` 를 사용하여 Uni를 Mono, Flux로 변환하여 반환하면 되겠습니다.  

## **마무리**  
기존의 JPA를 그리워 하면서 Webflux - Spring Data R2DBC 를 사용하고 계셨던 분들에게는 희소식이지 않을까 싶습니다. 또한 기존 JPA 스펙과 크게 다르지 않은 것으로 보아하니 기존의 JPA와 non-blocking 코드에 익숙하신 분들이라면 쉽게 익힐수 있을 것이라 생각합니다.  

저도 'Webflux는 쓸만한 orm 라이브러리 나오면 제대로 파야지~' 라고 생각하곤 했었는데 이제 제대로 한번 공부해봐야 겠습니다.  

## **참조**  
- [https://hibernate.org/reactive/documentation/1.1/reference/html_single/](https://hibernate.org/reactive/documentation/1.1/reference/html_single/)  
- [https://itnext.io/integrating-hibernate-reactive-with-spring-5427440607fe?gif=true](https://itnext.io/integrating-hibernate-reactive-with-spring-5427440607fe?gif=true)




