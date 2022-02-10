---
title: Hibernate Reactive 
author: keencho
date: 2022-02-10 10:12:00 +0900
categories: [Java]
tags: [Java, Hibernate, Webflux]
---

# **개요**
[Hibernate Reactive](https://hibernate.org/reactive/)가 드디어 정식 출시 되었습니다. (~~사실 현재기준 3달 지났습니다.~~)  

현재는 `Quarkus`, `Panache`와 사용했을때 더 효율성을 발휘하는 것으로 보입니다. 그래도 예전 `Spring Data JPA`가 나오기전 Hibernate를 사용할때를 떠올리며 내용을 정리해 보았습니다. 언젠가는 `Spring Data Reactive JPA(?)`가 나오길 희망해 봅니다.  

## **Hibernate Reactive**  
Webflux가 나온지 4년이 지났습니다. `Spring Data R2DBC` 를 통해 non-blocking 하게 데이터베이스에 접근할 수 있지만, 이렇다할 ORM 구현체는 없는 상황이었습니다.  

Hibernate Reactive는 Hibernate ORM 을 위한 리액티브 API로, 데이터베이스에 non-blocking하게 접근할 수 있도록 지원합니다. Hibernate Reactive는 데이터베이스와 non-blocking 방식으로 통신하도록 설계된 최초의 ORM 구현체 입니다.  

또한 많은 부분이 기존의 Hibernate ORM, JPA 2.2 과 동일하기 때문에, 만약 JPA에 대한 이해도가 깊은 분이라면 쉽게 Hibernate Reactive에 적응하실 것이라 생각합니다.

## **소개**  
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
    <persistence-unit name="test">
        <provider>org.hibernate.reactive.provider.ReactivePersistenceProvider</provider>

        <properties>

            <!-- PostgreSQL -->
            <property name="javax.persistence.jdbc.url"
                      value="jdbc:postgresql://localhost:5432/hibernate_reactive"/>

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

참고: [https://hibernate.org/reactive/documentation/1.1/reference/html_single/](https://hibernate.org/reactive/documentation/1.1/reference/html_single/)
