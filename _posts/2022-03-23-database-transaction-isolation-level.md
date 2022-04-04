---
title: Transaction Isolation
author: keencho
date: 2022-03-23 20:12:00 +0900
categories: [Database]
tags: [Database, Transaction]
---

# 트랜잭션
트랜잭션 관리는 데이터베이스와 통신하는 코드를 작성하는 백엔드 개발자라면 누구에게나 중요한 항목입니다. 트랜잭션의 옵션에 따라 성능이 크게 향상될수 있고 메소드 내부에서 일어나는 예외 상황이 달라질수 있기 때문입니다. 따라서 트랜잭션을 이해하고 설정하는 것이 정말 중요하다고 할수 있습니다.  

### Isolation level option    
Isolation level (격리수준) 은 트랜잭션의 주요 속성(원자성, 일관성, 격리, 내구성) 중 하나입니다. 동시에 여러 트랜잭션이 처리될 때 한 트랜잭션이 다른 트랜잭션에서 변경하거나 조회하는 데이터를 볼수 있게 허용할지 말지 결정하는 것입니다.  

예를들어 Spring 에서는 5가지 Isolation level 을 세팅할수 있습니다. 만약 그중 하나인 `DEFAULT` 를 사용한다면 사용하고 있는 데이터베이스의 기본 옵션을 따라갑니다.`PostgreSQL, Oracle, SQL Server`의 경우 `READ_COMMITTED`을 사용하고, `MySQL`의 경우 `REPEATABLE_READ`을 사용합니다.  

격리 수준이 허용하는 옵션들에 대해 먼저 알아보죠.

##### 1. Dirty read  
첫번째는 Dirty read(더티 리드) 입니다. 더티 리드는 하나의 트랜잭션이 아직 커밋되지 않은 다른 트랜잭션의 변경점을 조회할 수 있음을 의미합니다.  

![dirty-read](/assets/img/custom/spring/jpa/transaction/dirty-read.jpg)  

트랜잭션A가 시작된 이후 트랜잭션B가 시작됩니다. 트랜잭션A는 id가 10인 엔티티를 insert 합니다. 그 후 바로 트랜잭션B는 id가 10인 엔티티를 조회합니다. 그런데 그 이후 트랜잭션A가 문제가 생겨 모두 롤백되어 버렸습니다. 이제 트랜잭션B 에서 조회된 entity 객체는 데이터 무결성을 보장받지 못하는 상태가 되었습니다.  

##### 2. Non-repeatable read  
두번째는 Non-repeatable read (반복 가능하지 않은 조회) 입니다. 반복 가능하지 않은 조회는 한 트랜잭션에서 동일한 행을 읽더라도 다른 트랜잭션에서 발생한 update or delete가 적용되기 때문에 행 내의 값이 다를수 있음을 의미합니다.  

![non-repeatable-read](/assets/img/custom/spring/jpa/transaction/non-repeatable%20read.jpg)  

트랜잭션 A가 시작된 이후 트랜잭션B가 시작됩니다. 트랜잭션A는 id가 10인 엔티티를 조회합니다. 그 후 트랜잭션B가 id가 10인 엔티티를 수정합니다. 그 후 다시 트랜잭션A가 id가 10인 엔티티를 조회합니다. 트랜잭션A는 동일한 행을 조회하였지만 그 각각의 결과값은 달라집니다.  

##### 3. Phantom read  
마지막 세번쨰는 Phantom read(팬텀 리드) 입니다. 팬텀 리드는 한 트랜잭션에서 동일한 조회 쿼리를 수행하더라도 다른 트랜잭션에서 발생한 insert가 적용되기 때문에 행의 갯수(열)가 다를 수 있음을 의미합니다.  

![phantom-read](/assets/img/custom/spring/jpa/transaction/phantom%20read.jpg)  

트랜잭션A는 entity 테이블에서 모든 row를 조회합니다. 그 후 트랜잭션B에서 id가 10인 엔티티를 추가하였습니다. 그 후 다시 트랜잭션A에서 entity 테이블의 모든 row를 조회한다면 row의 갯수가 한개 늘어났을 것입니다.

##### Non-repeatable read vs Phantom read
반복 가능하지 않은 조회와 팬텀 리드 간에는 차이가 있습니다. 예제를 통해 알아보도록 하겠습니다. 

1. 트랜잭션 A시작
2. 트랜잭션 B시작
3. 트랜잭션 A에서 X 쿼리(조회) 수행
4. 트랜잭션 B에서 Y 쿼리(INSERT, UPDATE) 수행 
5. 트랜잭션 B 커밋
6. 트랜잭션 A에서 X 쿼리(조회) 수행

위와같은 플로우로 한 싸이클이 돈다고 가정해 보았습니다. 만약 `Non-repeatable read` 수준이라면 트랜잭션 B에서 INSERT가 일어나더라도 3번의 결과 행의 갯수와 6번의 결과 행의 갯수는 동일할 것입니다. 만약 UPDATE가 일어났다면 행의 갯수는 동일하지만 각각 행의 값은 다르겠지요.  

만약 `Phantom read` 수준이라면 트랜잭션 B에서 INSERT가 일어나면 3번의 결과 행의 갯수와 6번의 결과 행의 갯수는 다를 것입니다. 하지만 UPDATE가 일어난다면 각각 행의 값은 동일하겠지요.  

### Isolation level  
옵션들에 대해 알아봤으므로 이제 격리 수준의 종류에 대해 정리해보겠습니다.  

|Isolation Level|Dirty Read|Non-repeatable read|Phantom read|
|:-----:|:-----:|:-----:|:-----:|  
|READ_UNCOMMITTED|O|O|O|
|READ_COMMITTED|X|O|O|
|REPEATABLE_READ|X|X|O|
|SERIALIZABLE|X|X|X|  

격리 수준을 정리한 표입니다. `READ_UNCOMMITTED` (레벨 0) 부터 `SERIALIZABLE` (레벨 3) 까지 확인하실 수 있습니다.  

##### Isolation level 선택하기  
격리 수준에 대한 조정은 동시성, 데이터 무결성과 연관되어 있습니다. 동시성을 증가시키면 (레벨 0) 데이터 무결성에 문제가 발생하고 데이터 무결성을 유지하면 (레벨 3) 동시성이 떨어져 발생하는 비용이 증가하기 때문에 실제 어플리케이션에 영향을 끼치겠지요.  

처리하는 로직이 모두 다르기 떄문에 특정한 격리 수준이 좋다 나쁘다를 말하기는 어렵겠지요. 따라서 로직에 따라 적절한 수준을 선택하는 것이 중요하다고 볼수 있겠습니다.  
