---
title: JPA EntityListeners 에 의존성 주입하기
author: keencho
date: 2022-03-17 20:12:00 +0900
categories: [JPA]
tags: [JPA, Hibernate]
---

# JPA EntityListeners
JPA를 사용하다보면 여러가지 상황(before insert, after insert, before update, after update...)을 캐치하여 작업을 해야할 경우가 생기곤 합니다. 그럴때 하이버네이트 에서 제공하는 EntityListeners를 사용하여 원하는 작업을 수행할 수 있습니다.  

![entity callback list](/assets/img/custom/spring/jpa/entitylisteners/img.png)  

위 캡쳐 이미지는 하이버네이트에서 지원하는 콜백 리스트입니다. 해당 콜백 메소드들을 사용해 간단히 엔티티 이벤트를 캐치할 수 있습니다. 짧은 코드로 구현하자면 다음과 같은 형태가 되겠지요.  

```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
@EntityListeners(value = SoccerTeamListeners.class)
@Table(name = "soccer_team")
public class SoccerTeam {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    Long id;

    String name;
}
```

```java
@Component
public class SoccerTeamListeners{

    @Autowired
    SoccerTeamRepository soccerTeamRepository;

    @PrePersist
    public void prePersist(SoccerTeam soccerTeam) {
        System.out.println("size: " + soccerTeamRepository.findAll().size());
    }
}

```

# JPA EntityListeners Dependency Injection
위의 예제 코드는 잘못된 코드입니다. `EntityListeners`로 지정한 클래스에서 의존성 주입이 필요해 `SoccerTeamRepository`를 주입하였지만, 정작 prePersist 메소드 수행시 soccerTeamRepository에는 null이 할당되어있을 것입니다. null이 할당되어 있으니 당연히 메소드 호출시 `NullPointerException` 이 발생합니다.  

에러가 발생하는 이유는 `EntityListeners`는 `Spring IOC` 에서 관리되는 클래스가 아니기 때문입니다. 정확히는 `EntityListeners`에 등록되는 클래스에 `@Component` 어노테이션을 붙이면 bean으로 등록되긴 하지만 JPA 관련 프로퍼티들이 bean으로 등록되고 난 후에 Spring 관련 빈들이 등록되기 때문입니다.  

따라서 Spring bean이 모두 등록된 후 `EntityListeners`로 지정한 클래스를 모두 찾아 의존성 주입을 하면 되겠네요. 인터넷을 검색해보면 많은 해결방법들이 있습니다. 저는 제 나름대로의 방법을 소개하고자 합니다.  

## Example Code  
```java
@Entity
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@EntityListeners(value = SoccerPlayerListeners.class)
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
엔티티입니다. `SoccerPlayerListeners.class`를 `EntityListeners`로 등록하였습니다.  

```java
public interface SimpleEventListeners {
}
```
구현할 메소드가 아무것도 없는 인터페이스입니다. 이 인터페이스는 추후 사용될 것입니다.  

```java
@Component
public class SoccerPlayerListeners implements SimpleEventListeners {

    private static SoccerPlayerRepository soccerPlayerRepository;

    @PrePersist
    public void prePersist(SoccerPlayer soccerPlayer) {
        System.out.println("total soccer player count: " + soccerPlayerRepository.findAll().size());
    }
}
```
`SoccerPlayerListeners` 클래스입니다. 구현할 메소드가 없긴 하지만 위에서 정의한 인터페이스를 구현하는 클래스입니다. 모든 객체가 메모리를 공유하도록 하기 위해 static 키워드를 사용하여 변수를 선언하였습니다.

```java
@Configuration
public class SimpleEventListenerConfig {

    @Autowired
    private ApplicationContext applicationContext;

    @Autowired
    private List<SimpleEventListeners> simpleEventListenerList;

    @PostConstruct
    public void injectDependency() {
        simpleEventListenerList.forEach(el -> {
            Arrays.stream(el.getClass().getDeclaredFields())
                    .filter(field -> Modifier.isStatic(field.getModifiers()))
                    .forEach(field -> {
                        field.setAccessible(true);
                        try {
                            field.set(field.getType(), applicationContext.getBean(field.getType()));
                        } catch (Exception ex) {
                            System.err.println(ex.getMessage());
                        }
                    });
        });
    }
}
```
가장 중요한 config 파일입니다. `@PostConstruct`가 실행될때는 모든 Spring 관련 bean들이 등록된 후입니다.  

여기서 `SimpleEventListeners` 인터페이스를 사용한 이유가 나옵니다. `SimpleEventListeners`를 구현하는 모든 클래스들을 변수에 할당해야하기 때문입니다. 이 방식은 `List<SimpleEventListeners>` 객체를 루프 돌면서 static 변수만 필터링하고 리플렉션을 이용해 static 변수에 값을 할당하는 방식입니다.  

그리고 나서 테스트해보면 정상적으로 수행되는것을 확인하실 수 있을 것입니다. (java 17에서도 문제없이 동작합니다.)
