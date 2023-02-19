---
title: 동일한 클래스 안에서 새로운 트랜잭션 생성하기
author: keencho
date: 2023-02-19 20:12:00 +0900
categories: [Spring]
tags: [Spring, JPA]
---

# **동일한 클래스 안에서 새로운 트랜잭션 생성하기**
동일한 클래스 내에서 새로운 트랜잭션을 만들어 예외를 회피하려고 하는 경우, 의도했던 것과는 다른 결과가 나올 수 있습니다.

```java
@Transactional
public void answer(Inquiry inquiry) {
    System.out.println(TransactionSynchronizationManager.getCurrentTransactionName());

    try {
        this.updateAnswer(inquiry);
    } catch (Exception ignored) { }

    try {
        this.updateAnswerer(inquiry);
    } catch (Exception ignored) { }
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void updateAnswer(Inquiry inquiry) {
    System.out.println(TransactionSynchronizationManager.getCurrentTransactionName());
    
    // update answer
}

@Transactional
public void updateAnswerer(Inquiry inquiry) {
    System.out.println(TransactionSynchronizationManager.getCurrentTransactionName());

    // update answerer
    // exception
}
``` 

예를들어 위 코드와 같이 `updateAnswer` 로직을 새로운 트랜잭션으로 처리한다고 가정합니다.
따라서 `updateAnswererAndThrowException` 메소드에서 예외가 발생하더라도 별도의 트랜잭션에서 update 작업이 수행되었기 때문에 값이 정상적으로 저장되었을 것이라 생각했습니다.

그러나 작업이 끝난 이후 조회해보니 값이 업데이트 되지 않은 것을 확인하였습니다. 아무리봐도 코드에 이상한점은 없어 `TransactionSynchronizationManager.getCurrentTransactionName()` 를 통해 각 메소드로 진입할때 트랜잭션 이름을 확인해보기로 하였습니다.

```
com.keencho.application.service.InquiryService.answer
com.keencho.application.service.InquiryService.answer
com.keencho.application.service.InquiryService.answer
```

`updateAnswer` 메소드의 전파레벨을 `REQUIRES_NEW`로 지정했음에도 새로운 트랜잭션이 시작되지 않았습니다.

# **원인**
원인은 [스프링 공식 문서](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction-declarative-annotations) 에 설명되어 있습니다.

> In proxy mode (which is the default), only external method calls coming in through the proxy are intercepted. This means that self-invocation (in effect, a method within the target object calling another method of the target object) does not lead to an actual transaction at runtime even if the invoked method is marked with @Transactional. Also, the proxy must be fully initialized to provide the expected behavior, so you should not rely on this feature in your initialization code — for example, in a @PostConstruct method.

문제는 Spring의 AOP 프록시는 확장되지 않고 외부에서의 호출을 가로채기 위해 서비스 인스턴스를 감싼다는 것입니다.
만약 서비스 내부에서 `this` 통해 다른 메소드를 호출한다면 이는 인스턴스에 의해 바로 호출되며 래핑 프록시는 이를 가로챌 수 없습니다. (당연히 프록시는 이러한 인스턴스 내부의 호출을 전혀 인식하지 못합니다.)

# **해결**
해결 방법은 다음과 같습니다.

1. Spring AOP 대신 AspectJ 사용
2. self-injection
3. 클래스 분리

AOP 라이브러리를 변경하는것은 주의가 필요합니다. 예상하는 결과와 다른 결과를 받을 수 있습니다.

self-injection은 클래스 내부에서 자기 자신을 또 주입하는 방법입니다.

```java
@Service
public class InquiryService {

    @Lazy
    @Autowired
    private InquiryService self;
    
    ...
}
```  

다만 bean 이름이 겹쳐 등록이 안될수도 있으니 위와같이 `@Lazy` 어노테이션과 함께 사용해야 합니다.

사실 새로운 트랜잭션이 필요한 경우 클래스를 분리하여 Spring AOP가 호출을 가로채 새로운 트랜잭션을 시작할 수 있도록 하는게 가장 좋은 방법입니다. 다만 클래스가 분리됨에 따라 관리 포인트가 조금 늘어날 수 있다는 단점이 있습니다.

그 외에도 람다를 사용한 [이런](https://stackoverflow.com/a/56327004/13160032) 방법도 사용할 수 있습니다.

어플리케이션의 전체적인 설계를 망치지 않는 범위에서 방법을 찾아 적용해 보시기 바랍니다.
