---
title: 커맨드 패턴 (Command Pattern)
author: keencho
date: 2021-07-08 20:12:00 +0900
categories: [Design Pattern]
tags: [Design Pattern, Java]
---

# **커맨드 패턴**  
위키에 따르면 커맨드 패턴은 요청을 객체의 형태로 캡슐화하여 사용자가 보낸 요청을 나중에 이용할 수 있도록 매서드 이름, 매개변수 등 요청에 필요한 정보를 저장 또는 로깅, 취소할 수 있게 하는 패턴이라고 정의되어 있습니다.

<br/>
## **자세히**
위의 설명만으로는 이해하기가 힘듭니다. 예제를 통해 알아가도록 하겠습니다.

> 저는 택배배송 어플리케이션을 개발하고 있습니다.  
> 택배 상태가 업데이트되면 각각 상태에 따라 수행해야 하는 액션이 다릅니다.

위와 같은 상황에 직면해 있을때 먼저 커맨드 패턴 미적용 코드부터 살펴보겠습니다.
```java
public class ActionHandler {
    public void action(Order order) {
        String message = null;
    
        if (order.getOrderStatus() == OrderStatus.RECEIPT) {
            message = "접수 완료 되었습니다.";
            dispatch(order); // 기사에게 배차
        } else if (order.getOrderStatus() == OrderStatus.DISPATCH) {
            sendPushToDriver(order); // 기사에게 배차되었음을 알리는 push 메시지 발송
        } else if (order.getOrderStatus() == OrderStatus.START) {
            message = "배송이 시작되었습니다.";
        } else if (order.getOrderStatus() == OrderStatus.COMPLETE) {
            message = "배송이 완료되었습니다.";
        }
    
        if (message != null) {
            sendMessage(order, message);
        }
    }
    
    private void dispatch(Order order) {
        // ... 배차 로직
        System.out.println(order.getToName() + " 고객에게 배송될 물건 배차 완료");
    }
    
    
    private void sendPushToDriver(Order order) {
        // ... 푸시 발송 로직
        System.out.println(order.getDriverName() + " 기사에게 푸시 발신성공");
    }
    
    private void sendMessage(Order order, String message) {
        String number = order.getToNumber();
        // ... 메시지 발송 로직
        System.out.println(number + " 에게 메시지 발신 성공");
    }
}
```
비즈니스 로직에 따라 택배 상태를 변경한 후 모든 택배상태의 후처리 작업을 한 메소드로 처리할수 있는 클래스입니다. 현재 택배 상태에 따라 각각 수행햐야 하는 작업이 다르며 조건문을 통해 각각의 작업을 수행합니다.
