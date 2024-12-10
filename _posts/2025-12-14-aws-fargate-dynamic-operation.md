---
title: 야간 및 주말에 AWS Fargate 동적으로 운영하기
author: keencho
date: 2024-12-14 08:12:00 +0900
categories: [AWS]
tags: [AWS, DevOps]
---

# **야간 및 주말에 AWS Fargate 동적으로 운영하기**
사내에서만 사용하는 관리자(관제) 서비스가 있다. 아주 가끔 업무 시간외에 쓸일이 있긴 하지만 대부분의 경우 평일 21시 ~ 08시, 주말에는 전혀 사용하지 않는다.

클라우드의 장점중 `필요한 만큼만 사용, 필요할 때만 사용`을 중요한 장점으로 생각한다. 그래서 이 기회에 AWS Fargate 를 동적으로 운영해보기로 했다.

## **현재 아키텍처**
현재 아키텍처는 다음과 같다.
![1](/assets/img/custom/aws-fargate-dynamic-operation/1.png)

CloudFront에서 1차적인 요청을 처리하고 경로가 `/api` 로 시작한다면 api endpoints로 간주하여 로드밸런서 - Fargate로 요청을 보내고 그 외는 frontend api로 간주하여 s3로 요청을 보낸다.
프론트는 React, 백엔드는 Spring Boot로 구성되어 있다.

## **업무 외 시간 아키텍처**
업무 외 시간 아키텍처는 다음과 같다.
![2](/assets/img/custom/aws-fargate-dynamic-operation/2.png)

프론트엔드로 가는 요청은 건들지 않았다. 아예 처음부터 `CloudFront` 도 업무 외 시간 트래픽을 핸들링할수 있게 별도의 배포를 구성할까도 고민해봤지만, CloudFront 배포에서는 동일한 대체 도메인 이름을 여러 배포에 중복해서 사용할 수 없기 때문에 포기했다.
예를들어 업무시간에는 a.b.c -> a 배포, 그 외 시간에는 a.b.c -> b 배포 이게 안된다. 물론 업무 외 시간에 a 배포의 대체 도메인을 바꿔버리면 되지만 CloudFront 수정 후 배포 시간이 꽤 소요되기 때문에 깔끔하게 포기했다.

백엔드 아키텍처를 변경했다. 기존 Fargate로 가는 요청을 Lambda로 보낸다. 그 외 `EventBridge` 가 수행하는 동작에 대해서는 아래에서 설명한다.

## **Lambda 함수 작성하기**
람다가 처리하는 일이 많다. 처리하는일은 다음과 같다.

- EventBridge 스케줄러 실행 요청 처리
- Application Load Balancer 로부터 전달된 요청 처리
- EventBridge 이벤트 규칙을 통해 전달된 이벤트 처리

다음은 람다의 index 코드이다. 여기서 분기를 통해 각 이벤트를 처리한다.
```javascript
export const handler = async (event, context) => {

  if (event) {
    // 로드밸런서 요청
    if (event.requestContext && event.requestContext.elb) {
      return handleAlb(event, context);
    }

    if (event.sourceLocation && event.sourceLocation === 'EventBridge') {
      // 서비스가 SERVICE_STEADY_STATE 가 되었을때
      if (event.eventName === 'SERVICE_STEADY_STATE') {
        return handleService(event, context);
      }

      // EventBridge 스케줄러 실행 요청 처리
      if (event.eventName === 'FARGATE_SCALING_TRIGGER') {
        return handleScaling(event, context);
      }
    }
  }

  return null;
};
```

다음은

### **EventBridge 스케줄러 실행 요청 처리**
첫번째는 EventBridge 스케줄러 실행 요청 처리이다. `Amazon EventBridge -> Scheduler -> 일정` 에서 설정하였으며 어떻게 설정하는지까지 설명하진 않는다. 나의 경우 30분마다 작동하게 설정하였으며, 다음과 같은 페이로드를 전달하도록 하였다.
```json
{ "sourceLocation": "EventBridge", "eventName": "FARGATE_SCALING_TRIGGER" }
```

위 페이로드를 전달받은 람다는 `handleScaling` 메소드를 호출한다. 이 메소드는 1차로 시간을 확인한다. 현재 시간이 주말이거나 평일 21시 ~ 08시 사이면 `stop` 메소드를, 그 외에는 `start` 메소드를 호출한다.
```javascript
export const handleScaling = async(event, context) => {
  const now = new Date();
  const utcHour = now.getUTCHours();
  const kstHour = (utcHour + 9) % 24

  const isNightTime = (kstHour >= 21) || (kstHour < 8);

  const kstTime = new Date(now.getTime() + (9 * 60 * 60 * 1000));
  const kstDay = kstTime.getUTCDay();
  const isWeekend = kstDay === 0 || kstDay === 6;

  for (const data of TARGET_ECS_DATA) {
    if (isNightTime || isWeekend) {
      await stop(data, true)
    } else {
      await start(data, context)
    }
  }
}
```


