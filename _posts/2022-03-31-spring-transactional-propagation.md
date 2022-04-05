---
title: Spring Transaction Propagation
author: keencho
date: 2022-03-31 08:12:00 +0900
categories: [Spring]
tags: [Spring, Transaction]
---

# 트랜잭션 전파레벨
Spring의 `@Transactional` 어노테이션에는 많은 옵션이 존재합니다. 저번에 알아본 [격리 레벨](https://keencho.github.io/posts/jpa-transaction-isolation-level/){:target="\_blank"} 도 그중 하나이지요. 이번에 알아볼 옵션은 `Propagation(전파레벨)` 옵션입니다.  

트랜잭션에서의 전파는 비즈니스 로직의 트랜잭션 경계를 정의합니다. 스프링은 개발자가 설정한 레벨에 따라 트랜잭션을 시작하고 중지합니다. 스프링은 `TransactionManager:getTransaction()`을 호출하여 전파레벨에 따라 트랜잭션을 가져오거나 만듭니다. 이는 Transaction Manager의 모든 유형의 전파레벨중 일부를 지원하지만, 특정한 구현체에서만 지원되는 전파레벨도 일부 있습니다.  

### 1. REQUIRED  
`REQUIRED`는 기본 전파레벨 입니다. 스프링은 활성화된 트랜잭션이 있는지 확인하고 만약 없다면 새로운 트랜잭션을 생성합니다. 존재하는 트랜잭션이 있다면 실행할 비즈니스 로직이 활성화된 트랜잭션에 추가됩니다.

```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
@Table(name = "soccer_player")
public class SoccerPlayer {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    Long id;

    String name;

    @ManyToOne
    SoccerTeam soccerTeam;

    @Override
    public String toString() {
        return String.format("name = %s / soccerTeamName = %s", name, soccerTeam != null ? soccerTeam.getName() : null);
    }
}
```

```java
@Repository
public interface SoccerPlayerRepository extends JpaRepository<SoccerPlayer, Long> {
}
```

```java
@Service
public class TransactionService {

    @Autowired
    SoccerPlayerRepository soccerPlayerRepository;

    @Autowired
    TransactionChildService transactionChildService;

    @Transactional
    public void required() {
        SoccerPlayer player1 = new SoccerPlayer();
        player1.setName("손흥민");

        try {
            transactionChildService.insert(player1);
        } catch (Exception ex) {
            ex.printStackTrace();
        }

        SoccerPlayer player2 = new SoccerPlayer();
        player2.setName("박지성");
        soccerPlayerRepository.save(player2);
    }
}
```  

```java
@Service
public class TransactionChildService {

    @Autowired
    SoccerPlayerRepository soccerPlayerRepository;

    @Transactional(propagation = Propagation.REQUIRED)
    public void insert(SoccerPlayer soccerPlayer) {
        soccerPlayerRepository.save(soccerPlayer);
        throw new RuntimeException("throw exception");
    }
}
```

```java
@SpringBootTest
public class TransactionPropagationTest {

    @Autowired
    TransactionService transactionService;

    @Autowired
    SoccerPlayerRepository soccerPlayerRepository;

    @Test
    public void REQUIRED() {
        transactionService.required();

        var list = soccerPlayerRepository.findAll();

        Assert.notNull(list, "soccer player list must not be null");
    }
}
```  

예제 코드입니다. 첫번째 예제라 앞으로의 테스트에 사용되는 entity, repository, service 및 test class 까지 모두 작성하였습니다.  

부모 메소드에서 자식 메소드를 호출하고 자식 메소드는 에러를 throw 하는 코드입니다. `transactionChildService.insert()` 를 try - catch 구문으로 감쌌기때문에 문제없는 코드라고 생각이 들 수도 있지만 막상 테스트 결과를 보면 `Transaction silently rolled back because it has been marked as rollback-only` 라는 에러를 던지고 테스트가 종료됩니다.  

`REQUIRED`의 핵심은 모든 트랜잭션이 하나로 연결되는 것이므로 모든 로직이 정상적으로 통과 되더라도 트랜잭션이 끝나는 시점에는 롤백이 되는 것입니다. 자세한 이유는 [이 포스트](https://keencho.github.io/posts/transaction-rollback/){:target="\_blank"}를 통해 확인해 보세요.  

### 2. SUPPORTS  
두번째는 `SUPPORTS` 전파레벨 입니다. 활성화된 트랜잭션이 있다면 그것을 사용하고, 만약 없다면 트랜잭션 없이 메소드가 수행됩니다.  

```java
@Test
public void SUPPORTS_1() {
    transactionService.supports_1();

    var list = soccerPlayerRepository.findAll();

    Assert.isTrue(list.size() == 10, "size of soccer player list must be 2");
}

@Test
public void SUPPORTS_2() {
    transactionService.supports_2();

    var list = soccerPlayerRepository.findAll();

    Assert.isTrue(list.size() == 10, "size of soccer player list must be 2");
}
```

```java
@Transactional
public void supports_1() {
    doSupportsTest();
}

public void supports_2() {
    doSupportsTest();
}

private void doSupportsTest() {
    for (int i = 0; i < 10; i ++) {
        try {
            long seq = (long) i + 1;
            var soccerPlayer = new SoccerPlayer();
            soccerPlayer.setId(seq);
            soccerPlayer.setName("선수" + seq);

            transactionChildService.insertSupports(soccerPlayer);
        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }
}
```

```java
@Transactional(propagation = Propagation.SUPPORTS)
public void insertSupports(SoccerPlayer soccerPlayer){
    if (soccerPlayer.getId() > 5L) {
        throw new RuntimeException("throw exception");
    }
    soccerPlayerRepository.save(soccerPlayer);
}
```  
예제 코드입니다. 테스트를 `SUPPORTS_1()`과 `SUPPORTS_2()`로 나눴습니다. 첫번재 테스트의 경우 부모 메소드에 `@Transactional`을 붙였고 두번째 테스트의 부모 메소드에는 붙이지 않았습니다.  

결론적으로 두 테스트 모두 실패하긴 하지만 실패 사유는 다릅니다. `@Transactional`을 붙인 테스트의 경우 insert 자체가 일어나지 않는데 반해 `@Transaction`을 붙이지 않은 테스트의 경우 축구선수 list의 size가 5가 되었습니다.  

첫번째 테스트의 경우 자식 메소드가 수행되는 시점에 활성화된 트랜잭션(부모 트랜잭션)이 존재하기 때문에 트랜잭션이 종료되는 시점에 수행된 모든 작업이 롤백되게 됩니다.  

두번째 테스트의 경우 활성화된 트랜잭션이 없기 때문에 non-transactional 하게 동작하게 되어 5개의 insert된 엔티티는 롤백되지 않게 되는 것이지요.  

### 3. MANDATORY  
세번째는 `MANDATORY` 입니다. 활성화된 트랜잭션이 있다면 그것을 사용하고, 활성화된 트랜잭션이 없다면 Exception이 발생합니다.  

```java
@Test
public void MANDATORY() {
    transactionService.mandatory();

    soccerPlayerRepository.findByName("손흥민").orElseThrow(() -> new RuntimeException("soccer player must not be null"));
}
```

```java
public void mandatory() {
    SoccerPlayer player1 = new SoccerPlayer();
    player1.setName("손흥민");
    transactionChildService.insertMandatory(player1);
}
```  

```java
@Transactional(propagation = Propagation.MANDATORY)
public void insertMandatory(SoccerPlayer soccerPlayer) {
    soccerPlayerRepository.save(soccerPlayer);
}
```

`insertMandatory()`가 수행되는 시점에 활성화된 부모 트랜잭션이 없기 떄문에 `org.springframework.transaction.IllegalTransactionStateException: No existing transaction found for transaction marked with propagation 'mandatory'` Exception이 발생하며 테스트가 종료됩니다.  

### 4. NEVER 
네번째는 `NEVER` 입니다. 활성화된 트랜잭션이 있다면 Exception이 발생합니다.  

```java
@Test
public void NEVER() {
    transactionService.never();

    soccerPlayerRepository.findByName("손흥민").orElseThrow(() -> new RuntimeException("soccer player must not be null"));
}
```

```java
@Transactional
public void never() {
    SoccerPlayer player1 = new SoccerPlayer();
    player1.setName("손흥민");
    transactionChildService.insertNever(player1);
}
```

```java
@Transactional(propagation = Propagation.NEVER)
public void insertNever(SoccerPlayer soccerPlayer) {
    soccerPlayerRepository.save(soccerPlayer);
}
```

`insertNever()`가 수행되는 시점에 활성화된 부모 트랜잭션이 있기 때문에 `org.springframework.transaction.IllegalTransactionStateException: Existing transaction found for transaction marked with propagation 'never'` Exception이 발생하며 테스트가 종료됩니다.  

### 5. NOT_SUPPORTED  
다섯번째는 `NOT_SUPPORTED` 입니다. 활성화된 트랜잭션 유무에 상관없이 non-transactional 하게 동작합니다. 활성화된 트랜잭션이 있는 경우 트랜잭션을 일시중지 시킵니다. 작업이 끝나면 자동으로 중지한 트랜잭션을 다시 시작합니다.  

### 6. NESTED
여섯번째는 `NESTED` 입니다. `NESTED`는 활성화된 트랜잭션의 **중첩** 트랜잭션을 시작합니다. 중첩 트랜잭션이 시작할 때 `SAVEPOINT`가 생성됩니다. 중첩 트랜잭션이 실패한다면 생성한 `SAVEPOINT`로 롤백합니다. (SAVEPOINT를 지원하는 DB만 사용 가능 - PostgreSQL, Oracle, MSSQL 등) 중첩 트랜잭션은 외부 트랜잭션의 일부이므로 외부 트랜잭션이 끝날 때만 커밋됩니다. 중첩 트랜잭션이 끝났다고 하여 바로 커밋되는 일은 없습니다.  

```java
@Test
public void NESTED() {
    transactionService.nested();

    var soccerPlayerList = soccerPlayerRepository.findAll();

    Assert.isTrue(soccerPlayerList.size() == 1, "size of soccer player list must be 1");
}
```  

```java
@Transactional
public void nested() {
    SoccerPlayer player1 = new SoccerPlayer();
    player1.setName("손흥민");
    soccerPlayerRepository.save(player1);

    SoccerPlayer player2 = new SoccerPlayer();
    player2.setName("박지성");
    soccerPlayerRepository.save(player2);
    transactionChildService.insertNested(player2);
}
```  

```java
@Transactional(propagation = Propagation.NESTED)
public void insertNested(SoccerPlayer soccerPlayer) {
    try {
        soccerPlayerRepository.save(soccerPlayer);
        throw new RuntimeException("throw error");
    } catch (Exception ex) {
        ex.printStackTrace();
    }
}
```  

예제 코드입니다. 박지성 이라는 이름을 가진 엔티티를 저장하는 시점에 예외가 발생하였지만 `SAVEPOINT`까지만 롤백되어 손흥민 이라는 이름을 가진 엔티티는 정상적으로 저장되게 되어 테스트를 통과하게 됩니다.  

### 7. REQUIRES_NEW
마지막 7번째는 `REQUIRES_NEW` 입니다. 아마 기본 `REQUIRED`와 함께 가장 많이 사용하게될 전파레벨이 될텐데요, `REQUIRES_NEW`는 활성화된 트랜잭션이 있다면 잠시 중지시키고 새로운 트랜잭션을 생성하여 진행합니다.  

```java
@Test
public void REQUIRES_NEW() {
    try {
        transactionService.requiresNew();
    } catch (Exception ex) {
        ex.printStackTrace();
    }

    var soccerPlayerList = soccerPlayerRepository.findAll();

    Assert.isTrue(soccerPlayerList.size() == 1, "size of soccer player list must be 1");
}
```

```java
@Transactional
public void requiresNew() {
    SoccerPlayer player1 = new SoccerPlayer();
    player1.setName("손흥민");
    transactionChildService.insertRequiresNew(player1);

    SoccerPlayer player2 = new SoccerPlayer();
    player2.setName("박지성");
    soccerPlayerRepository.save(player2);
    throw new RuntimeException("throw error on parent");
}
```

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void insertRequiresNew(SoccerPlayer soccerPlayer) {
    soccerPlayerRepository.save(soccerPlayer);
}
```  

예제 코드입니다. 부모 트랜잭션에서 RuntimeException을 던졌지만 자식 트랜잭션은 새로운 트랜잭션으로 작업을 수행하였기 때문에 부모 트랜잭션에 영향을 받지 않아 손흥민 이라는 이름을 가진 엔티티는 정상적으로 저장이 되게 되어 테스트를 통과하게 됩니다.  


