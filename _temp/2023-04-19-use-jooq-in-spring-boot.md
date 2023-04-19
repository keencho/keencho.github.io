---
title: Spring Boot에서 JOOQ 사용하기
author: keencho
date: 2023-04-19 08:12:00 +0900
categories: [JOOQ]
tags: [JOOQ, Spring Boot]
---

# **JOOQ**

## **JOOQ란?**
JOOQ(Java Oriented Querying)는 Java를 사용하여 안전한 SQL 쿼리를 작성할 수 있도록 하는 데이터베이스 쿼리 프레임워크 입니다. 개발자가 SQL 쿼리를 보다 직관적이고 자연스럽게 작성할 수 있도록 하며 type-safe한 API를 제공합니다. 이를 통해 컴파일시 오류를 발견 / 수정할 수 있습니다.

JOOQ는 PostgreSQL, Oracle, MySQL과 같은 관계형 데이터베이스들을 지원합니다. 다만 NoSQL 데이터베이스는 기본적으로 지원하지 않습니다. 일부 제한적인 기능을 제공하긴 하는데 그럴빠에야 `Spring Data MongoDB` 와 같이 해당 영역에 특화된 프레임워크를 사용하는게 좋습니다.

JOOQ는 단순 쿼리 작업 외에도 데이터베이스 스키마 생성, 코드 생성 및 기타 데이터베이스 관련 작업을 지원하므로 Java에서 데이터베이스 기반 애플리케이션을 구축하는데 유용한 도구입니다.

## **JOOQ vs Querydsl**
JOOQ과 Querydsl은 개발자가 type-safe한 SQL 쿼리를 작성할 수 있도록 한다는 공통점이 있습니다. 그러나 두 프레임워크 사이에는 몇가지 차이점이 있습니다.

1. DSL 구문
   - JOOQ는 SQL 구문을 기반으로 DSL을 사용합니다.
   - Querydsl은 Java 구문을 기반으로 하는 DSL을 사용합니다.
2. 지원 범위
   - JOOQ는 관계형 데이터베이스들을 중점으로 지원합니다.
   - Querydsl은 관계형 데이터베이스와 비관계형 데이터베이스(NoSQL)을 모두 지원합니다.
3. 코드 생성
   - JOOQ는 데이터베이스 스키마를 기반으로 Java 클래스를 생성합니다.
   - QueryDSL은 자바 코드를 바탕으로 Java 클래스를 생성합니다.

## **그래서 둘중 무엇을 선택해야 하는가?**
사실 개인적인 생각으론 JOOQ와 Querydsl를 비교하는건 잘못됐다고 생각합니다. 어떤 Persistence Framework를 선택했느냐에 따라 달라지기 때문이죠.

스프링 기반 웹 어플리케이션을 개발할때 관계형 데이터베이스를 사용한다면 Persistence Framework로 ORM 혹은 SQL Mapper를 선택합니다. ORM의 경우 `Hibernate - Spring Data JPA - Querydsl`조합을 사용하고 SQL Mapper의 경우 `MyBatis`를 사용합니다.
SQL Mapper 를 선택했다면 JOOQ는 MyBatis 대신 사용할 수 있는 프레임워크라고 할 수 있습니다. XML 기반인 Mybatis와 다르게 Java로 type-safe하게 쿼리를 작성할 수 있기 때문이죠.

따라서 만약 개발중인 어플리케이션이 MyBatis를 사용하고 type-safe하게 쿼리를 작성할 수 있는 프레임워크를 찾고있다면 JOOQ는 완벽한 프레임워크라고 볼 수 있습니다. ORM을 사용한다면 비즈니스 로직에는 `Hibernate - Spring Data JPA - Querydsl`조합을 사용하고 통계성 데이터나 대용량 배치 작업을 할때, 혹은 native query가 필요할 때 JOOQ를 사용하는 것이 좋다고 생각합니다.

# **Spring Boot에서 JOOQ 사용하기**
본론으로 들어와 Spring Boot 에서 JOOQ를 사용하는 방법에 대해 알아보도록 하겠습니다.

빌드 도구는 maven을 사용하며 db는 PostgreSQL을 사용합니다.
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jooq</artifactId>
        <version>3.0.5</version>
    </dependency>

    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.6.0</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <version>3.0.5</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

제일 처음으로 테이블을 생성합니다.
```sql
CREATE TABLE author (
                        id             INT          NOT NULL PRIMARY KEY,
                        first_name     VARCHAR(50),
                        last_name      VARCHAR(50)  NOT NULL
);

CREATE TABLE book (
                      id             INT          NOT NULL PRIMARY KEY,
                      title          VARCHAR(100) NOT NULL
);

CREATE TABLE author_book
(
    author_id INT NOT NULL,
    book_id   INT NOT NULL,

    PRIMARY KEY (author_id, book_id),
    CONSTRAINT fk_ab_author FOREIGN KEY (author_id) REFERENCES author (id)
        ON UPDATE CASCADE ON DELETE CASCADE,
    CONSTRAINT fk_ab_book FOREIGN KEY (book_id) REFERENCES book (id)
);
```

테이블을 생성했다면 spring datasource를 정의합니다. 만약 방언을 지정하지 않는다면 `DEFAULT` 를 사용합니다.
```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/jooq
spring.datasource.username=username
spring.datasource.password=password
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jooq.sql-dialect= postgres

logging.level.org.jooq.tools.LoggerListener=DEBUG
```

이제 maven 플러그인을 통해 생성한 테이블을 기반으로 자바 클래스를 생성할 것입니다.
```xml
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>properties-maven-plugin</artifactId>
    <version>1.1.0</version>
    <executions>
        <execution>
            <phase>initialize</phase>
            <goals>
                <goal>read-project-properties</goal>
            </goals>
            <configuration>
                <files>
                    <file>src/main/resources/application.properties</file>
                </files>
            </configuration>
        </execution>
    </executions>
</plugin>
<plugin>
    <groupId>org.jooq</groupId>
    <artifactId>jooq-codegen-maven</artifactId>
    <executions>
        <execution>
            <id>generate-postgres</id>
            <phase>generate-sources</phase>
            <goals>
                <goal>generate</goal>
            </goals>
            <configuration>
                <!-- JDBC connection parameters -->
                <jdbc>
                    <url>${spring.datasource.url}</url>
                    <user>${spring.datasource.username}</user>
                    <password>${spring.datasource.password}</password>
                    <driver>${spring.datasource.driver-class-name}</driver>
                </jdbc>
                <!-- Generator parameters -->
                <generator>
                    <database>
                        <name>
                            org.jooq.meta.postgres.PostgresDatabase
                        </name>
                        <includes>.*</includes>
                        <excludes/>
                        <inputSchema>public</inputSchema>
                        <includeSystemSequences>true</includeSystemSequences>
                    </database>
                    <generate>
                        <pojos>true</pojos>
                        <pojosEqualsAndHashCode>true</pojosEqualsAndHashCode>
                        <javaTimeTypes>true</javaTimeTypes>
                        <fluentSetters>true</fluentSetters>
                        <sequences>true</sequences>
                    </generate>
                    <target>
                        <packageName>com.keencho.jooq.model</packageName>
                        <directory>target/generated-sources/jooq</directory>
                    </target>
                </generator>
            </configuration>
        </execution>
    </executions>
</plugin>
```
프로퍼티에 접근하기 위해 `properties-maven-plugin`을 사용합니다. `jooq-codegen-maven` 플러그인을 사용하며 설정에 jdbc 정보, db명, 옵션, 타겟 패키지, 타겟 디렉토리 등 필요한 정보들을 입력합니다.

필요한 정보를 모두 입력 했다면 `mvn clean generate-sources` 명령을 수행합니다. 아래처럼 클래스가 생성되었다면 성공입니다.

![generate-source-1](/assets/img/custom/use-jooq-in-spring-boot/generate-source-1.JPG)

