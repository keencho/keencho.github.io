---
title: Transaction silently rolled back because it has been marked as rollback-only
author: keencho
date: 2021-07-28 20:12:00 +0900
categories: [Spring]
tags: [Spring, Transaction]
---

## **이게 왜 저장이 안될까?**
서비스를 운영하다가 이 포스팅의 제목과 같이 `Transaction silently rolled back because it has been marked as rollback-only` 라는 에러 메시지를 받게 되었습니다.
해당 에러의 원인은 금방 찾았으나, 한 서비스가 내부 서비스의 메소드를 try-catch 블록으로 감싸 호출하는 형태로 코드를 작성했기 때문에 예외가 발생하더라도 commit은 정상적으로 이루어질 것으로 예상했습니다.

그러나 제 마음처럼 되지 않았고 예외 발생전 작성한 저장 / 수정 코드가 롤백 되어 위의 에러 메시지를 받게 되었습니다. 이 포스트에서는 왜 이런일이 발생한 것인지, 해결 방법은 있는 것인지 알아보도록 하겠습니다.

## **상황 재현하기**
상용 서비스와 비슷한 환경을 만들어 상황을 재현해보겠습니다. 현재 서비스는 spring-boot 2.4.2, postgresql, spring data jpa 를 중심으로 이루어져 있습니다.
```java
@Service
@Transactional
@Slf4j
public class AccountParentService {

    @Autowired
    AccountChildService accountChildService;

    public void saveAccountWithException() {
        try {
            accountChildService.saveAccountAndThrowException();
        } catch (Exception ex) {
            log.error("내부 서비스에서 계정 생성중 에러 발생 - " + ex.getMessage());
        }

        log.info("계정 생성 완료");
    }
}
```

```java
@Service
@Transactional
public class AccountChildService {

    @Autowired
    AccountRepository accountRepository;

    public void saveAccountAndThrowException() {
        accountRepository.save(
                Account.builder()
                        .name("이름")
                        .loginId("로그인 ID")
                        .age(30)
                        .password(UUID.randomUUID().toString())
                        .build()
        );

        throw new RuntimeException("예외 던지기");
    }
}
```
앞서 말씀드렸다시피 외부 트랙잭션 클래스의 메소드가 try-catch 블록으로 내부 트랜잭션 클래스의 메소드를 감싼 채로 호출하고 있습니다. 실제 `throw new RuntimeException()`를 던졌다 하더라도 `accountRepository.save()` 이후이고
외부 메소드에서는 예외처리가 되어있기 때문에 계정이 생성된다고 생각할 수 있지만 실제로는 ***'계정 생성 완료'*** 로그 이후에 아래와 같은 에러메시지가 출력되고 있었습니다.

```
2021-08-24 10:49:31.131 ERROR 16632 --- [nio-9999-exec-2] s.s.basic.service.AccountParentService   : 내부 서비스에서 계정 생성중 에러 발생 - 예외 던지기
2021-08-24 10:49:31.131  INFO 16632 --- [nio-9999-exec-2] s.s.basic.service.AccountParentService   : 계정 생성 완료
2021-08-24 10:49:31.148 ERROR 16632 --- [nio-9999-exec-2] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is org.springframework.transaction.UnexpectedRollbackException: Transaction silently rolled back because it has been marked as rollback-only] with root cause

org.springframework.transaction.UnexpectedRollbackException: Transaction silently rolled back because it has been marked as rollback-only
	at org.springframework.transaction.support.AbstractPlatformTransactionManager.processCommit(AbstractPlatformTransactionManager.java:752) ~[spring-tx-5.3.9.jar:5.3.9]
	at org.springframework.transaction.support.AbstractPlatformTransactionManager.commit(AbstractPlatformTransactionManager.java:711) ~[spring-tx-5.3.9.jar:5.3.9]
	at org.springframework.transaction.interceptor.TransactionAspectSupport.commitTransactionAfterReturning(TransactionAspectSupport.java:654) ~[spring-tx-5.3.9.jar:5.3.9]
	at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:407) ~[spring-tx-5.3.9.jar:5.3.9]
	at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:119) ~[spring-tx-5.3.9.jar:5.3.9]
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186) ~[spring-aop-5.3.9.jar:5.3.9]
	at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:750) ~[spring-aop-5.3.9.jar:5.3.9]
	at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:692) ~[spring-aop-5.3.9.jar:5.3.9]
	at sycho.spring.basic.service.AccountParentService$$EnhancerBySpringCGLIB$$b70115de.saveAccountWithException(<generated>) ~[main/:na]
...
<생략>
```

## **원인 파악하기**
스택 트레이스를 참조하여 롤백이 진행되는 메소드를 찾아보니 `TransactionAspectSupport` 클래스의 `completeTransactionAfterThrowing` 메소드를 찾을수 있었습니다.
```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        if (logger.isTraceEnabled()) {
            logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
                    "] after exception: " + ex);
        }
        if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
            try {
                txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
            }
            catch (TransactionSystemException ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                ex2.initApplicationException(ex);
                throw ex2;
            }
            catch (RuntimeException | Error ex2) {
                logger.error("Application exception overridden by rollback exception", ex);
                throw ex2;
            }
        }
...
<생략>
```

다시 `txtInfo.transactionAttriute.rollbackOn(ex)` 메소드를 찾아 들어가 보면 `DefaultTransactionAttribute` 클래스의 `rollBackOn()` 메소드를 만날수 있습니다.
```java
@Override
public boolean rollbackOn(Throwable ex) {
    return (ex instanceof RuntimeException || ex instanceof Error);
}
```
아까 던진 예외는 `RuntimeException` 이기 때문에 롤백이 발생하게 되는군요. 다시 롤백 프로세스를 살펴보겠습니다.
```java
if (status.hasTransaction()) {
    if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
        if (status.isDebug()) {
            logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
        }
        doSetRollbackOnly(status);
    }
    else {
        if (status.isDebug()) {
            logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
        }
    }
}
```
`processRollback()` 메소드를 살펴보면 `status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()` 을 만족하면 `rollbackOnly` 를 세팅하게 되어 있습니다.
여차저차 프로세스를 따라가보면 `TransactionDriverControlImpl` 의 `rollbackOnly` 필드를 true로 변경하는 것을 확인할 수 있습니다.
근데 이 트랜잭션 에서는 `rollbackOnly` 필드만 변경할 뿐 내부 클래스의 트랜잭션에서는 `UnexpectedRollbackException` 를 바로 던지지는 않네요.
```java
AbstractPlatformTransactionManager.processRollback()

if (unexpectedRollback) { // false
    throw new UnexpectedRollbackException(
            "Transaction rolled back because it has been marked as rollback-only");
}
```

내부 클래스의 트랜잭션이 완료되면 최초 트랜잭션 처리가 시작됩니다. 이번에는 `status.isNewTransaction()` 조건을 만족하고 트랙잭션의 상태가 `MARKED_ROLLBACK` 으로 마크되어있기 때문에 `status.isGlobalRollbackOnly()` 가 true를 return 함에 따라 `unexpectedRollback` 필드가 true가 됩니다.
```java
else if (status.isNewTransaction()) {
    if (status.isDebug()) {
        logger.debug("Initiating transaction commit");
    }
    unexpectedRollback = status.isGlobalRollbackOnly();
    doCommit(status);
}
```
그렇다면 최종적으로 아래 코드에 의해 `UnexpectedRollbackException`이 발생하는 것을 확인할 수 있습니다.
```java
// Throw UnexpectedRollbackException if we have a global rollback-only
// marker but still didn't get a corresponding exception from commit.
if (unexpectedRollback) {
    throw new UnexpectedRollbackException(
            "Transaction silently rolled back because it has been marked as rollback-only");
}
```

## **그럼 왜?**
결론적으로 보면 `status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()` 조건중 `isGlobalRollbackOnParticipationFailure()` 이 true 이면 전체 롤백이 되도록 되어있는것 같네요.

해당 메소드에 들어가서 이것저것 살펴보고 있는데 마침 위에 `setGlobalRollbackOnParticipationFailure()` 메소드의 주석에 딱 제가 원하는 내용이 있었습니다.
```
/**
 * Set whether to globally mark an existing transaction as rollback-only
 * after a participating transaction failed.
 * <p>Default is "true": If a participating transaction (e.g. with
 * PROPAGATION_REQUIRED or PROPAGATION_SUPPORTS encountering an existing
 * transaction) fails, the transaction will be globally marked as rollback-only.
 * The only possible outcome of such a transaction is a rollback: The
 * transaction originator <i>cannot</i> make the transaction commit anymore.
 * <p>Switch this to "false" to let the transaction originator make the rollback
 * decision. If a participating transaction fails with an exception, the caller
 * can still decide to continue with a different path within the transaction.
 * However, note that this will only work as long as all participating resources
 * are capable of continuing towards a transaction commit even after a data access
 * failure: This is generally not the case for a Hibernate Session, for example;
 * neither is it for a sequence of JDBC insert/update/delete operations.
 * <p><b>Note:</b>This flag only applies to an explicit rollback attempt for a
 * subtransaction, typically caused by an exception thrown by a data access operation
 * (where TransactionInterceptor will trigger a {@code PlatformTransactionManager.rollback()}
 * call according to a rollback rule). If the flag is off, the caller can handle the exception
 * and decide on a rollback, independent of the rollback rules of the subtransaction.
 * This flag does, however, <i>not</i> apply to explicit {@code setRollbackOnly}
 * calls on a {@code TransactionStatus}, which will always cause an eventual
 * global rollback (as it might not throw an exception after the rollback-only call).
 * <p>The recommended solution for handling failure of a subtransaction
 * is a "nested transaction", where the global transaction can be rolled
 * back to a savepoint taken at the beginning of the subtransaction.
 * PROPAGATION_NESTED provides exactly those semantics; however, it will
 * only work when nested transaction support is available. This is the case
 * with DataSourceTransactionManager, but not with JtaTransactionManager.
 * @see #setNestedTransactionAllowed
 * @see org.springframework.transaction.jta.JtaTransactionManager
 */
```
번역기의 힘을 빌려 대충 번역하여 다음과 같은 정보를 얻을수 있었습니다.
 - 기본 트랜잭션을 rollback-only 로 마킹하는 것을 전역적으로 적용할지 말지 세팅
 - 기본값은 true
 - 만약 참여중인 트랜잭션의 타입이 PROPAGATION_REQUIRED 거나 PROPAGATION_SUPPORTS 이고 실패하면 트랜잭션은 전역적으로 rollback-only로 마킹된다.
 - rollback-only로 마킹된 트랜잭션은 결국 롤백되게 된다.

사실 왜 이렇게 세팅되어 있는지는 정확히는 잘 모르겠습니다만, 저는 롤백 조건중 `excpetion instanceof RuntimeException` 이라는 조건이 있는것으로 보았을때 런타임 시점에 Unchecked Exception 이 발생하면
그것을 감싸고 있는 외부의 예외처리 블록이 있더라도 내부 메소드는 롤백을 원하기 때문에 커밋을 원하는 외부 메소드와 충돌(모순?) 이 일어나 결론적으로는 모두 롤백된다. 라고 마음속으로 결론을 내리겠습니다.

## **해결 방법**
이 글을 작성하면서 드는 생각입니다만, 편하다고 해서 비즈니스 로직에 무턱대고 예외를 던지는 행위는 또다른 문제를 일으킬수 있을것 같다는 생각이 들었습니다. 예외 객체를 만드는것 자체가 다른 객체를 만드는 비용에 비해 비싸기도 하지요.

물론 해결방법은 있습니다. 상황에 알맞은 방법으로 문제를 해결해 보시기 바랍니다.

#### 1. Checked Exception

`RuntimeException`이 아닌 `Exception` 을 상속받아 CustomException을 만들고 예외를 던져야 할 때 이 CustomException을 던지면 롤백되지 않습니다.
```java
public class CustomException extends Exception {
    public CustomException(String msg) { super(msg); }
}
```

```java
public void saveAccountAndThrowException() throws CustomException {
    accountRepository.save(
            Account.builder()
                    .name("이름")
                    .loginId("로그인 ID")
                    .age(30)
                    .password(UUID.randomUUID().toString())
                    .build()
    );

    throw new CustomException("예외 던지기");
}
```
이유는 `Exception`을 상속받아 만든 Exception은 Checked Exception 이기 때문입니다. 간단하게 `Exception`을 상속받아 만든 Exception은 Checked Exception 이고 `RuntimeException`을 상속받아 만든
Exception은 Unchecked Exception 이라고 생각하면 될 것 같습니다.

위에서 살펴봤다시피 롤백 조건중 `excpetion instanceof RuntimeException` 조건이 있었습니다. 반대로 말하면 ***'Checked Exception은 발생하더라도 롤백되지 않는다.'*** 가 됩니다.

Checked Exception은 복구가 가능하다는 전제조건을 가지고 있기 때문에 예외가 발생하더라도 복구가 가능할 때 사용하는 것이 좋습니다. 하지만 실질적으로 모든 경우에 대하여 복구 전략을 세우기 어렵기 때문에
정말 확실한 경우에만 사용하는 편이 좋겠지요.

<br/>
#### 2. noRollbackFor 속성 사용
@Transactional 메소드에는 `noRollbackFor` 속성이 있습니다. (javax 패키지의 @Transactional 에는 dontRollbackOn 속성이 있습니다.) 이 속성을 이용하면 예외가 발생해도 롤백하지 않을 예외 클래스를 지정할 수 있습니다.
단, 이 속성에 들어갈 클래스는 `Throwable` 의 서브클래스여야 합니다.

```java
@org.springframework.transaction.annotation.Transactional(noRollbackFor = RuntimeException.class)
@javax.transaction.Transactional(dontRollbackOn = RuntimeException.class)
public void saveAccountAndThrowException() {
    accountRepository.save(
            Account.builder()
                    .name("이름")
                    .loginId("로그인 ID")
                    .age(30)
                    .password(UUID.randomUUID().toString())
                    .build()
    );

    throw new RuntimeException("예외 던지기");
}
```

<br/>
#### 3. Propagation.REQUIRES_NEW
일단 재현 코드를 바꿔보겠습니다.

```java
@Service
@Slf4j
public class AccountParentService {

    @Autowired
    AccountRepository accountRepository;

    @Autowired
    AccountChildService accountChildService;

    @Transactional
    public void saveAccountWithException() {
        try {
            accountChildService.saveAccountAndThrowException();
        } catch (Exception ex) {
            log.error("내부 서비스에서 계정 생성중 에러 발생 - " + ex.getMessage());
        }

        accountRepository.save(
                Account.builder()
                        .name("테스트 이름")
                        .loginId("테스트 ID")
                        .age(20)
                        .password(UUID.randomUUID().toString())
                        .build()
        );

        log.info("계정 생성 완료");
    }
}
```

```java
@Service
public class AccountChildService {

    @Autowired
    AccountRepository accountRepository;

    @Transactional
    public void saveAccountAndThrowException() {
        accountRepository.save(
                Account.builder()
                        .name("이름")
                        .loginId("로그인 ID")
                        .age(30)
                        .password(UUID.randomUUID().toString())
                        .build()
        );

        throw new RuntimeException("예외 던지기");
    }
}
```
내부 메소드에서 계정 생성 이후 외부 메소드로 다시 돌아와 또 다른 계정을 생성하는 코드를 작성하였습니다. 당연히 현재는 내부 트랜잭션이 전파되기 때문에 전역 롤백이 발생합니다.

그렇다면 다음과 같이 내부 메소드의 @Transactional 전파옵션을 REQUIRES_NEW 로 변경해 보겠습니다.
```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAccountAndThrowException() {
...
<생략>
```
다시 작업을 수행해 보니 내부 메소드의 작업은 롤백되었지만 외부 메소드의 작업은 정상적으로 진행되어 계정 한개가 생성되었습니다.

이 글에서 자세히 다루지는 않을 것이지만 `내부 메소드에 진입시 새로운 트랜잭션을 시작하여 외부 트랜잭션과 다른 독립적인 트랙잭션으로 작동하기 때문에 내부에서 예외가 발생해도 외부로는 롤백이 전파되지 않는다.` 라고 이해하시면 될 것 같습니다.
단, 이 전략은 개발자가 원치않은 또다른 끔찍한 에러를 발생시킬 수 있기 때문에 트랜잭션의 동작 원리 / 방식에 대한 **명확한** 이해가 필요합니다.

<br/>
## **결론**
의도치 않은 롤백 상황이 일어나는 이유와 해결방법에 대해 알아보았습니다.
막상 일이 일어났을때는 조금 당황했지만 이유를 알고보니 정말 멋지고 안전한 전략이라는 생각이 들었습니다.
스프링의 동작방식에 대해 하나둘 알아가면 갈수록 참 재밌는 것 같습니다. ~~동시에 좌절감을 주기도 하지요..~~

해결 방법에 대해서는 몇가지 적긴 했지만 저는 비즈니스 로직에 예외를 발생시켜야 하는 경우 해당 예외를 `Exception`을 상속받은 Checked Exception으로 만들고,
확실한 복구 전략을 세워 명확한 예외처리를 하는 방법을 추천드립니다.
