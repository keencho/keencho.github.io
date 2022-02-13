---
title: 옵저버 패턴 (Observer Pattern)
author: keencho
date: 2021-07-09 20:12:00 +0900
categories: [Design Pattern]
tags: [Design Pattern, Java]
---

# **옵저버 패턴**
옵저버 패턴은 객체의 상태 변화를 관찰하는 관찰자들, 즉 옵저버의 목록을 객체에 등록하여 상태 변화가 있을 때마다 메스드 등을 통해 객체가 직접 목록의 각 옵저버에게 통지하도록 하는 디자인 패턴입니다.

## **구조**
![structure](/assets/img/custom/design-pattern/observer/structure.png)

1. 게시자는 다른 객체들에게 관심 이벤트를 발행합니다. 이 이벤트난 게시자가 상태를 변경하거나 어떤 동작을 실행할 때 발생합니다.
게시자에는 새 구독자가 가입하고 현재 구독자가 목록에서 나갈 수 있도록 하는 구독 인프라가 포함되어 있습니다.
2. 새 이벤트가 발생하면 게시자는 구독 목록을 살펴보고 각 구독차 객체의 인터페이스에 선언된 알림 메소드를 호출합니다.
3. 구독자는 구독 인터페이스를 선언합니다. 대부분의 경우 단일 update 방법으로 구성됩니다. 메소드에는 게시자가 업데이트와 함께 일부 이벤트 세부 정보를 전달할 수 있도록 하는 매개 변수가 있을 수 있습니다.
4. 구독자는 게시자가 발행한 알림에 대해 여러가지 작업을 수앻합니다. 게시자가 구현클래스와 결합되지 않도록 모든 클래스는 동일한 인터페이스로 구현되어야 합니다.
5. 대부분의 경우 구독자는 업데이트를 올바르게 처리하기 위해 전후 정보가 필요합니다. 이러한 이유로 게시자는 알림 메소드의 인수로 전후 정보를 전달하곤 합니다.
게시자는 자기 자신을 인자로 전달하여 구독자가 필요한 데이터를 직접 가져올 수 있습니다.
6. 클라이언트는 게시자 및 구독자 객체를 별도로 만든 후 게시자 업데이트를 위해 구독자를 등록합니다.

## **적용 가능한 경우**
1. 한 객체의 상태를 변경하기 위해 다른 객체를 변경해야할 경우 혹은 실제 객체 집합을 미리 알 수 없거나 동적으로 변경되는 경우 옵저버 패턴을 사용합니다.
2. 앱이 한정된 시간, 특정한 케이스에만 다른 객체를 관찰해야 하는 경우 옵저버 패턴을 사용합니다.

## **장단점**
#### **장점**
1. 게시자의 코드를 변경하지 않고 새로운 구독자 클래스를 만들어 작업할 수 있으므로 개방/폐쇄 원칙을 만족합니다.
2. 런타임 시점에 객체와 관계를 맺을 수 있습니다.

#### **단점**
1. 구독자는 랜덤으로 알림을 받습니다.
  - 우선순위를 지정하는 코드를 삽입하여 순위를 지정할수도 있겠지만, 복잡성과 결합성만 높아지기 때문에 추천하지는 않습니다.

## **예제**
기본적인 형태의 게시자부터 만들어 보겠습니다.
```java
public class EventManager {
    Map<String, List<EventListener>> listeners = new HashMap<>();

    public EventManager(String... operations) {
        for (String operation : operations) {
            this.listeners.put(operation, new ArrayList<>());
        }
    }

    public void subscribe(String eventType, EventListener listener) {
        List<EventListener> users = listeners.get(eventType);
        users.add(listener);
    }

    public void unsubscribe(String eventType, EventListener listener) {
        List<EventListener> users = listeners.get(eventType);
        users.remove(listener);
    }

    public void notify(String eventType, File file) {
        List<EventListener> users = listeners.get(eventType);
        for (EventListener listener : users) {
                listener.doAction(eventType, file);
        }
    }
}
```
위 코드는 게시자입니다. 구독자를 listners 변수로 받고 구독, 구독해지, 알림의 기능을 메소드로 가지고 있습니다.

```java
public class Editor {
    public EventManager events;
    private File file;

    public Editor() {
        this.events = new EventManager("open", "save");
    }

    public void openFile(String filePath) {
        this.file = new File(filePath);
        events.notify("open", file);
    }

    public void saveFile() throws Exception {
        if (this.file != null) {
            events.notify("save", file);
        } else {
            throw new Exception("파일을 먼저 열어주세요.");
        }
    }
}
```
위 코드는 클라이언트가 사용할 메소드를 모아놓은 게시자 클래스입니다. 파일을 열고 저장하는 기능을 내포하고 있습니다.

```java
public interface EventListener {
    void update(String eventType, File file);
}

public class LogOpenListener implements EventListener {
    private File log;

    public LogOpenListener(String fileName) {
        this.log = new File(fileName);
    }

    @Override
    public void update(String eventType, File file) {
        System.out.println(log + " 파일을 열었고 로그에 추가하였습니다.");
    }
}

public class EmailNotificationListener implements EventListener {
    private String email;

    public EmailNotificationListener(String email) {
        this.email = email;
    }

    @Override
    public void update(String eventType, File file) {
        System.out.println(eventType + " 으로 인해 " + email + " 로 이메일을 전송하였습니다.");
    }
}
```
위 코드는 구독자 코드들입니다. 옵저버 인터페이스가 있고 옵저버 인터페이스를 구현하는 클래스 2개가 있습니다.

```java
public class Client {
    public static void main(String[] args) {
        Editor editor = new Editor();
        editor.events.subscribe("open", new LogOpenListener("D:\\file.txt"));
        editor.events.subscribe("save", new EmailNotificationListener("admin@example.com"));

        try {
            editor.openFile("test.txt");
            editor.saveFile();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
마지막으로 클라이언트 코드입니다. 에디터를 생성하고 파일을 open 할 경우 LogOpenListener을 통해 해당 파일이 열렸다는 정보를 로그에 저장합니다.
파일을 save할 경우 파일이 저장되었다는 정보를 미리 지정해둔 이메일(admin@example.com) 에 전송합니다.

### **결론**
옵저버 패턴은 주로 gui 프로그램에서 사용되곤 합니다. 중요한점은 한 객체가 변경되었음을 다른 객체에 알려야 하는 경우 게시자(Publisher), 구독자(Observer), 알림(Notify)의 형태로 구현한다는 점입니다.

### **참조**
> [https://refactoring.guru/design-patterns/observer](https://refactoring.guru/design-patterns/observer)
> [https://refactoring.guru/design-patterns/observer/java/example](https://refactoring.guru/design-patterns/observer/java/example)
