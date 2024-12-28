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

> :exclamation: 본 포스팅에서 소개하는 로직은, 제 개인적인 접근 방식이므로 실제 상용 시스템에 적용하기 전 철저한 테스트 과정이 필요합니다.

## **현재 아키텍처**
현재 아키텍처는 다음과 같다.
![1](/assets/img/custom/aws-fargate-dynamic-operation/1.png)

CloudFront에서 1차적인 요청을 처리하고 경로가 `/api` 로 시작한다면 api endpoints로 간주하여 로드밸런서 - Fargate로 요청을 보내고 그 외는 frontend api로 간주하여 s3로 요청을 보낸다.
프론트는 React, 백엔드는 Spring Boot로 구성되어 있다.

## **업무 외 시간 아키텍처**
업무 외 시간 아키텍처는 다음과 같다.
![2](/assets/img/custom/aws-fargate-dynamic-operation/2.png)

프론트엔드로 가는 요청은 건들지 않았다. 아예 처음부터 `CloudFront` 도 업무 외 시간 트래픽을 핸들링할수 있게 별도의 배포를 구성할까도 고민해봤지만, CloudFront 배포에서는 동일한 대체 도메인 이름을 여러 배포에 중복해서 사용할 수 없기 때문에 포기했다.
예를들어 업무시간에는 a.b.c -> a 배포, 그 외 시간에는 a.b.c -> b 배포 이게 안된다. 물론 업무 외 시간에 a 배포의 대체 도메인을 바꿔버리면 되지만 CloudFront 수정 후 배포 시간이 꽤 소요되기 때문에 포기했다.

백엔드 아키텍처를 변경했다. 기존 Fargate로 가는 요청을 Lambda로 보낸다. 그 외 `EventBridge` 가 수행하는 동작에 대해서는 아래에서 설명한다.

## **Lambda 함수 작성하기**
람다가 처리하는 일이 많다. 처리하는일은 다음과 같다.

- EventBridge 스케줄러 실행 요청 처리
- EventBridge 이벤트 규칙을 통해 전달된 이벤트 처리
- Application Load Balancer 로부터 전달된 요청 처리

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

다음은 시스템을 핸들링할때 사용될 서비스 데이터이다. 사실 호스트나 기타 데이터로 필요한 arn들을 역으로 찾아갈수도 있겠으나 그렇게 되면 불필요한 호출이나 로직이 들어가기 때문에 필요한 정보들을 변수에 담아두기로 했다.
```javascript
const SERVICE_DATA = [
    {
        key: 'admin',
        host: 'admin.keencho.com',
        cluster: 'keencho-cluster',
        service: 'admin',
        albArn: '로드 밸런서 arn',
        albRuleArns: ['실제 admin.keencho.com 을 HTTP 호스트 헤더 조건으로 가지고 있는 리스너 규칙 arn 리스트'],
        lambdaTargetGroupArn: '람다가 포함되어 있는 대상그룹 arn',
        ecsFargateTargetGroupArn: ['실제 fargate 태스크가 포함되어 있는 대상그룹 arn 리스트']
    }
]
```

### **EventBridge 스케줄러 실행 요청 처리**
첫번째는 EventBridge 스케줄러 실행 요청 처리이다. `Amazon EventBridge -> Scheduler -> 일정` 에서 설정하였으며 어떻게 설정하는지까지 설명하진 않는다. 나의 경우 30분마다 작동하게 설정하였으며, 다음과 같은 페이로드를 전달하도록 하였다.
```json
{ "sourceLocation": "EventBridge", "eventName": "FARGATE_SCALING_TRIGGER" }
```

위 페이로드를 전달받은 람다는 `handleScaling` 메소드를 호출한다. 이 메소드는 1차로 시간을 확인한다. 현재 시간이 주말이거나 평일 21시 ~ 08시 사이면 `stop` 메소드를, 그 외에는 `start` 메소드를 호출한다.
```javascript
const handleScaling = async(event, context) => {
  const now = new Date();
  const utcHour = now.getUTCHours();
  const kstHour = (utcHour + 9) % 24

  const isNightTime = (kstHour >= 21) || (kstHour < 8);

  const kstTime = new Date(now.getTime() + (9 * 60 * 60 * 1000));
  const kstDay = kstTime.getUTCDay();
  const isWeekend = kstDay === 0 || kstDay === 6;

  for (const data of SERVICE_DATA) {
    if (isNightTime || isWeekend) {
      await stop(data)
    } else {
      await start(data, context)
    }
  }
}
```

##### **1. stop**
멈춰야 하는 시간대이면 멈춘다. 다음은 사용되는 전체 코드이다.
```javascript
// 서비스 가져오기
const getService = async(ecsClient, clusterName, serviceName) => {
  const response = await ecsClient.send(new DescribeServicesCommand({
    cluster: clusterName,
    services: [serviceName]
  }))

  if (!response || !response.services || response.services.length !== 1) {
    return undefined
  }

  return response.services[0];
}

// 트래픽 검증
const checkTraffic = async(data) => {
  const now = new Date();

  const end = new Date(Date.UTC(
    now.getUTCFullYear(),
    now.getUTCMonth(),
    now.getUTCDate(),
    now.getUTCHours(),
    now.getUTCMinutes(),
    now.getUTCSeconds()
  ));

  // 20분
  const start = new Date(end.getTime() - 20 * 60 * 1000);

  // 간격: 20분
  const period = 1200;

  const loadBalancerArn = data.albArn

  const loadBalancerName = loadBalancerArn.split(':').pop().split('/').slice(1).join('/');

  const sum = await data.ecsFargateTargetGroupArn.reduce(async (acc, arn) => {
    const accumulated = await acc;

    const tgName = arn.split(':').pop()

    const params = {
      StartTime: start,
      EndTime: end,
      MetricDataQueries: [
        {
          Id: "count",
          MetricStat: {
            Metric: {
              Namespace: "AWS/ApplicationELB",
              MetricName: "HTTPCode_Target_2XX_Count", // 200번대 상태 코드만 필터링
              Dimensions: [
                {
                  Name: "LoadBalancer",
                  Value: loadBalancerName
                },
                {
                  Name: "TargetGroup",
                  Value: tgName
                }
              ]
            },
            Period: period,
            Stat: "Sum" // 요청 개수의 합계
          },
          Label: "200 Status Code Requests"
        }
      ]
    };

    const cloudwatch = new CloudWatchClient({ region: 'ap-northeast-2' });
    const command = new GetMetricDataCommand(params);
    const response = await cloudwatch.send(command);

    const results = response.MetricDataResults[0]

    const trafficSum = (!results.Values || results.Values.length === 0) ? 0 : results.Values.reduce((sum, number) => sum + number, 0)

    return accumulated + trafficSum;

  }, Promise.resolve(0))

  return sum === 0
}

const stop = async(data) => {
  ////////// 1. 멈춰야 하는 서비스인지 검증
  const ecsClient = new ECSClient({region: 'ap-northeast-2'});

  const service = await getService(ecsClient, data.cluster, data.service);
  if (service === undefined || service.desiredCount === 0) {
    return
  }

  // 새벽에 도는 스케쥴의 경우 현재 desiredCount가 1 이상인 경우 지난 20분간 2xx 상태코드를 가진 트래픽이 있는지 검증한다.
  // 사용중인데 꺼지면 안되기 때문이다.
  if (!await checkTraffic(data)) {
    return;
  }

  ////////// 2. 멈춰야 하는 서비스라면 해당 서비스로 라우팅중인 로드밸런서 룰 변경
  const albClient = new ElasticLoadBalancingV2Client({region: 'ap-northeast-2'})
  const describeResponse = await albClient.send(
    new DescribeRulesCommand({
      RuleArns: data.albRuleArns
    })
  );

  for (const rule of (describeResponse.Rules || [])) {
    if (rule.Actions && rule.Actions.length === 1 && rule.Actions[0].Type === 'forward') {
      const action = rule.Actions[0]

      const updatedAction = {
        ...action,
        TargetGroupArn: data.lambdaTargetGroupArn,
        ForwardConfig: {
          ...action.ForwardConfig,
          TargetGroups: [
            {
              TargetGroupArn: data.lambdaTargetGroupArn,
              Weight: 1
            }
          ]
        }
      };

      await albClient.send(
        new ModifyRuleCommand({
          RuleArn: rule.RuleArn,
          Actions: [updatedAction]
        })
      )
    }
  }

  ////////// 3. 서비스 desired count 0 처리
  const params = {
    cluster: data.cluster,
    service: data.service,
    desiredCount: 0
  }

  await ecsClient.send(new UpdateServiceCommand(params));
}
```

순서는 다음과 같다.
1. 멈춰야 하는 서비스인지 검증한다.
- 이미 ecs 서비스의 desired count가 0 이면 수행하지 않는다.
- 지난 20분간 2xx 상태코드를 리턴한 요청이 있으면 수행하지 않는다. (이 스케줄러는 30분 간격으로 도는데 도는 시점에 사용하고 있을수도 있기 때문에)
2. `실제 admin.keencho.com 을 HTTP 호스트 헤더 조건으로 가지고 있는 리스너 규칙 arn 리스트` 리스너 규칙의 작업을 `람다 대상그룹으로 전달` 로 변경한다.
3. ecs 서비스의 desired count를 0으로 변경한다.

이렇게 하면 api 트래픽은 람다로 가고 서비스의 태스크 수는 0이 된다. 태스크가 동작하지 않으므로 당연히 요금이 부과되지 않는다.

##### **2. start**
업무시간이 다가오면 시작한다. 다음은 사용되는 전체 코드이다.
```javascript
const start = async(data, context) => {
  ////////// 1. 시작해야 하는 서비스인지 검증
  const ecsClient = new ECSClient({ region: 'ap-northeast-2' });

  const service = await getService(ecsClient, data.cluster, data.service);
  if (service === undefined || service.desiredCount !== 0) {
    return
  }

  ////////// 2. desired count 1 처리
  const params = {
    cluster: data.cluster,
    service: data.service,
    desiredCount: 1
  };

  await ecsClient.send(new UpdateServiceCommand(params));

  ////////// 3. EventBridge에 규칙 등록 (이미 등록되어 있는 경우 업데이트됨)
  const ruleName = `ecs-monitor-${data.cluster}_${data.service}`
  const ruleParams = {
    Name: ruleName,
    EventPattern: JSON.stringify({
      source: ["aws.ecs"],
      "detail-type": ['ECS Service Action'],
      resources: [service.serviceArn],
      detail: {
        eventName: ['SERVICE_STEADY_STATE']
      }
    }),
  };

  const eventBridgeClient = new EventBridgeClient({ region: "ap-northeast-2" });
  const result = await eventBridgeClient.send(new PutRuleCommand(ruleParams));

  ////////// 4. 규칙에 대한 타겟 설정
  const targetParams = {
    Rule: ruleName,
    Targets: [
      {
        Id: `${ruleName}-target`,
        Arn: context.invokedFunctionArn,
        Input: JSON.stringify({
          sourceLocation: 'EventBridge',
          eventName: 'SERVICE_STEADY_STATE',
          clusterArn: service.clusterArn,
          serviceArn: service.serviceArn,
          albArn: data.albArn,
          key: data.key
        })
      }
    ]
  }

  await eventBridgeClient.send(new PutTargetsCommand(targetParams));

  ////////// 5. lambda에 리소스 액세스 권한 부여
  try {
    const lambdaClient = new LambdaClient({ region: "ap-northeast-2" });
    const addPermissionParams = {
      Action: "lambda:InvokeFunction",
      FunctionName: context.functionName,
      Principal: "events.amazonaws.com",
      SourceArn: result.RuleArn,
      StatementId: `${ruleName}-permission`
    };

    await lambdaClient.send(new AddPermissionCommand(addPermissionParams));
  } catch (error) {
    // 권한이 이미 존재하는 경우 에러 발생 (StatementId 중복)
    if (error.name === 'ResourceConflictException') {
      // ...
    }
  }
}
```

순서는 다음과 같다.
1. 멈춰야 하는 서비스인지 검증한다.
- 이미 ecs 서비스의 desired count가 1 이면 수행하지 않는다.
2. ecs 서비스의 desired count를 1으로 변경한다.
3. `EventBridge`에 규칙을 등록한다.
- detail type: ECS Service Action
- event type: SERVICE_STEADY_STATE [참조](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/ecs_service_events.html)
4. 등록한 규칙에 대한 타겟을 설정한다. 람다와 Input 데이터를 지정한다.
5. `EventBridge` 규칙이 람다에 접근할 수 있도록 액세스 권한을 부여한다.
- 이 작업은 최초 1회만 하면 되며 그 이후 동일한 작업 수행시 `ResourceConflictException` 에러가 발생한다.

이렇게 하면 ecs 서비스 태스크 수가 1이 된다. 그러나 실제 로드밸런서 리스너 규칙을 변경하지는 않는다. 물론 이 시점에서 폴링을 통해 태스크 상태를 검증하는 방법도 있지만 람다는 실행시간 만큼 요금이 부과되기 때문에 ecs 서비스가 안정적인 상태에 도달하면 람다를 다시 호출하는 이벤트를 `EventBridge`에 등록하는 방식을 선택했다.

### **EventBridge 이벤트 규칙을 통해 전달된 이벤트 처리**
ecs 서비스의 태스크 수를 1로 변경하고 서비스가 안정적인 상태에 도달한다면, EventBridge에 등록해둔 `SERVICE_STEADY_STATE` 이벤트가 트리거되어 람다를 호출한다. 다음은 이를 핸들링하는 코드이다.
```javascript
const handleService = async(event) => {

  const data = SERVICE_DATA.find(item => item.key === event.key)

  if (!data) {
    return;
  }

  const input = {
    services: [event.serviceArn],
    cluster: event.clusterArn
  }

  const ecsClient = new ECSClient({ region: 'ap-northeast-2' });
  const command = new DescribeServicesCommand(input);
  const response = await ecsClient.send(command);

  if (!response.services || response.services.length === 0) {
    return;
  }

  const service = response.services[0];

  if (service.desiredCount === 0) {
    return;
  }

  if (!service.loadBalancers || service.loadBalancers.length === 0) {
    return;
  }

  const targetChangeArns = data.albRuleArns
  const targetGroupArn = service.loadBalancers[0].targetGroupArn;

  const albClient = new ElasticLoadBalancingV2Client({ region: 'ap-northeast-2' })

  const describeResponse = await albClient.send(
    new DescribeRulesCommand({
      RuleArns: targetChangeArns
    })
  );

  if (!describeResponse.Rules || describeResponse.Rules.length === 0) {
    return;
  }

  const rule = describeResponse.Rules[0];

  // 이미 라우팅이 정상인 경우 -> 배포의 경우
  if (rule.Actions.some(item => item.TargetGroupArn === targetGroupArn)) {
    return;
  }

  const updatedActions = rule.Actions.map(action => {
    return {
      ...action,
      TargetGroupArn: targetGroupArn,
      ForwardConfig: {
        ...action.ForwardConfig,
        TargetGroups: [
          {
            TargetGroupArn: targetGroupArn,
            Weight: 1
          }
        ]
      }
    };
  });

  for (const ruleArn of targetChangeArns) {
    await albClient.send(
      new ModifyRuleCommand({
        RuleArn: ruleArn,
        Actions: updatedActions
      })
    )
  }
}
```

순서는 다음과 같다.
1. 넘어온 페이로드 의 key로 `SERVICE_DATA` 에서 필요한 데이터를 찾는다.
2. ecs 서비스의 desired count가 0 이면 수행하지 않는다.
3. 이미 리스너 규칙이 올바른 대상 그룹으로 전달되고 있을 경우 수행하지 않는다.
- 서비스가 배포되는 경우도 이 람다가 실행되기 때문에 추가된 조건이다.
4. 람다 대상그룹을 바라보고 있는 리스너 규칙을 ecs 서비스 태스크가 포함된 대상그룹으로 변경한다
- 나의 경우 `CodeDeploy` 기반 배포를 진행하고 있고, 이는 2개의 대상그룹을 필요로 한다. 따라서 변경할 태스크 대상그룹은 하드코딩된 대상그룹이 아닌 ecs 서비스와 엮여있는 대상그룹으로 해야 한다.
- `Rolling Update` 기반 배포의 경우 대상그룹이 1개일 것이기 때문에 하드코딩된 대상그룹을 사용해도 무방하다.

### **Application Load Balancer 로부터 전달된 요청 처리**
지금까지의 작업만 진행하면 업무 외 시간에는 서비스의 필요 태스크가 0이 되고 업무 시간이 다가오면 서비스의 필요 태스크가 1이 된다. 하지만 만약에 긴급하게 서비스를 이용해야할 경우는 어떻게 해야 할까? 이에 대해 내가 생각한 방법을 작성해보도록 한다.


##### **1. 요청 검증하기**
앞서 서비스를 stop 할때, 리스너 규칙이 바라볼 대상을 람다로 변경했다. 업무 외 시간 긴급하게 서비스를 이용해야할 경우 이 람다로 요청이 전달될 것이다. 이 경우 서비스를 다시 시작해야 한다. 다만 2가지 고려사항이 있다.
1. 프로그램(봇) 에 의해 찔러본 요청인가?
2. 실제 서비스 (Spring Boot) 에 존재하는 경로인가?

1번의 경우 `CloudFront` 앞단에 `WAF` 를 배치하여 봇을 컨트롤 할 수 있다. 람다에서 직접 컨트롤 할 수도 있겠지만 내 경우 그냥 `WAF` 로 컨트롤하는 쪽을 선택했다.

2번의 경우 생각을 좀 해봤는데 내가 생각한 방법은 다음과 같다.
1. `@PostConstruct`를 이용하여 di가 끝난 후 앱 내에 있는 모든 경로를 검색해서 s3에 배열로 저장한다.
2. 실제 람다가 호출되었을때 배열에 path와 일치하는 요소가 존재하는 경우만 서비스를 시작한다.

다음은 모든 경로를 검색해서 s3에 업로드하는 코드이다. s3util 부분만 실제 올리는 로직으로 변경하면 될 것이다.
```java
@Profile("prod")
@Component
public class EndpointGenerator {

    @Autowired
    RequestMappingHandlerMapping requestMappingHandlerMapping;

    @Autowired
    S3Util s3Util;

    @Autowired
    ObjectMapper objectMapper;

    @Autowired
    WebApplicationContext webApplicationContext;

    @PostConstruct
    public void init() throws JsonProcessingException {

        var patterns = new ArrayList<String>();
        var contextPath = webApplicationContext.getServletContext().getContextPath();

        for (var info : requestMappingHandlerMapping.getHandlerMethods().keySet()) {
            var patternWithContext = info.getPatternValues().stream()
                    .map(pattern -> contextPath + pattern)
                    .collect(Collectors.toList());

            patterns.addAll(patternWithContext);
        }

        s3Util.upload(
                "keencho-bucket",
                "admin.json",
                    objectMapper.writeValueAsString(patterns)
                );
    }
}
```

##### **2. 시작하기**
이제 실제 시작에 필요한 코드를 작성해보자. 로드밸런서의 요청을 처리해야 하기 때문에 그에 맞는 형식을 리턴할 필요가 있다.
```javascript
const buildResponse = (success, data) => {
  return {
    headers: {
      "Content-Type": "application/json",
    },
    statusCode: success ? 200 : 500,
    body: JSON.stringify(data)
  }
}

const errorResponse = () => buildResponse(false, { success: false, message: 'Invalid access' })

const successResponse = () => buildResponse(true, { success: true })

const handleAlb = async(event, context) => {
  /////////////////////////////////////// 헬스 체크
  if (event.path === '/health-check') {
    return {
      statusCode: 200,
      statusDescription: "OK",
      isBase64Encoded: false,
      headers: {
        "Content-Type": "application/json"
      }
    };
  }

  try {
    return handle(event, context)
  } catch (error) {
    return errorResponse()
  }

}

const isValidRoute = async(type, url) => {

  const patternToRegex = (pattern) => {
    const regexPattern = pattern
      .replace(/\//g, '\\/') // 슬래시 이스케이프
      .replace(/\*\*/g, '.*') // ** 패턴을 .* (모든 문자, 여러 개)로 변환
      .replace(/{([^}]+)}/g, '([^/]+)'); // {변수} 패턴을 캡처 그룹으로 변환

    return new RegExp(`^${regexPattern}$`);
  }

  const client = new S3Client({});

  const command = new GetObjectCommand({
    Bucket: 'keencho-bucket',
    Key: type + '.json'
  })

  const responseData = await client.send(command);
  const str = await responseData.Body.transformToString();
  const routes = JSON.parse(str);

  for (const route of routes) {
    const regex = patternToRegex(route);
    if (regex.test(url)) {
      return true;
    }
  }

  return false;
}

const handle = async(event, context) => {

  const path = event.path;
  const host = event.headers.host;
  const data = SERVICE_DATA.find(item => item.host === host);

  if (!data) {
    return errorResponse()
  }

  const type = data.key

  ////////// 1. 검증
  if (!type || !await isValidRoute(type, path) || !event?.headers?.referer?.includes(host)) {
    return errorResponse()
  }

  ////////// 2. desired count 이미 1 이상인지 확인
  const ecsClient = new ECSClient({region: 'ap-northeast-2'});
  const service = await getService(ecsClient, data.cluster, data.service);
  if (service === undefined) {
    return errorResponse()
  }

  // 이미 시작된 경우라고 간주함.
  if (service.desiredCount > 0) {
    return successResponse(type)
  }

  ////////// 3. desire task 1 처리
  await start(data, context)

  return successResponse(type)
}
```

순서는 다음과 같다.
1. 데이터를 검증한다.
- `isValidRoute()` 메소드를 통해 유효한 경로인지 검증한다. `@PathVariable`를 사용한 경로일 수 있으므로 그에대한 정규식이 필요하다.
- 동일 출처인지 검증한다. 어짜피 따로 허용하는 코드를 작성하지 않았으므로 에러가 발생할 테지만 명시적으로 검증한다.
2. ecs 서비스의 desired count가 0보다 크면 수행하지 않는다.
3. 앞서 작성한 `start()` 함수를 호출한다.
4. 성공 응답을 리턴한다.

이미 검증을 했으므로 `start()` 메소드에서 또 검증할 필요는 없다. 불필요한 호출을 막는 로직을 추가하면 좋을것 같다. (여기선 하지 않는다.)

이후 태스크가 시작되면 EventBridge에 등록해둔 `SERVICE_STEADY_STATE`가 트리거되고 앞서 살펴본 [이벤트 처리 로직](#eventbridge-이벤트-규칙을-통해-전달된-이벤트-처리) 에 의해 처리된다.

## **결론**
현재 사내 시스템(관리자)과 테스트서버 에만 위 아키텍처를 적용하고 있다. 테스트서버의 경우 새벽시간대에 시작하는 로직이 빠져있다. 그냥 무조건 업무시간에만 쓰도록 한 것이다.

코드나 로직은 조금 보완해야 겠지만 예상대로 돌아감에 만족한다. 추가로 이 포스팅에 작성하진 않았지만 중지 시간대에 다음과 같이 UI가 구성되어 있으면 좋을것이다.
![3](/assets/img/custom/aws-fargate-dynamic-operation/3.png)

내 경우 서비스 유지기간인 경우 아예 별도의 페이지로 redirect 시키고 시작해야할 서비스면 서비스를 시작시키고 상태를 `START`, `PROCESSING`, `COMPLETED` 3단계로 간단하게 나누어 확인할 수 있도록 하고 시작하지 않아야 할 서비스면 위와 같은 화면이 보이도록 UI를 구성했다.

[Instance Scheduler](https://aws.amazon.com/ko/solutions/implementations/instance-scheduler-on-aws/) 이라는 솔루션이 존재하긴 하는데 이는 ECS Fargate 대상으로는 자동화 못하는거 같아 `EventBridge + Lambda` 조합으로 직접 구현해 봤다. 덕분에 리소스 비용이 조금이나마 절감되어 좋은듯 하다.
