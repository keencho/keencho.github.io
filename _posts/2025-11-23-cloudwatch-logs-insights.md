---
title: CloudWatch Logs Insights 로 Lambda 로그 파보기
author: keencho
date: 2025-11-23 17:52:00 +0900
categories: [AWS]
tags: [AWS]
---

# **CloudWatch Logs Insights 로 Lambda 로그 파보기**
Lambda를 운영하다 보면 로그를 봐야 할 일이 계속 생긴다. "어제 밤에 왜 에러가 났지", "이 함수 콜드스타트가 얼마나 되지", "특정 요청 하나가 어디서 막혔지" 같은거다. 근데 CloudWatch 콘솔에서 로그 그룹 들어가서 로그 스트림을 하나하나 열어보는건 정말 못할 짓이다. 스트림이 수십개로 쪼개져 있고, 시간순도 아니고, 원하는 한 줄을 찾으려면 한참을 스크롤해야 한다.

그래서 쓰는게 CloudWatch Logs Insights다. 로그를 SQL 비슷한 쿼리로 검색하고 집계할수 있다. 한번 손에 익으니 콘솔에서 눈으로 로그 뒤지던 시절로 못 돌아가겠더라.

## **쿼리는 파이프라인이다**
Logs Insights 쿼리는 파이프(`|`)로 명령을 이어붙이는 구조다. 왼쪽에서 오른쪽으로 로그가 흘러가면서 걸러지고(filter), 쪼개지고(parse), 집계된다(stats). 유닉스에서 명령어를 파이프로 잇는것과 똑같은 감각이다.

제일 기본은 에러 찾기다.

~~~
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50
~~~

`@timestamp` 와 `@message` 는 Logs Insights가 자동으로 잡아주는 필드다. 위 쿼리는 메시지에 ERROR가 들어간 로그만 걸러서, 최신순으로 50개 보여달라는 뜻이다. 콘솔에서 스트림 뒤지던걸 이 네 줄이 대신한다.

## **콜드스타트 들여다보기**
Lambda 운영하면서 자주 궁금한게 콜드스타트다. 함수가 한동안 안 불리다가 새로 뜰때 초기화 시간이 붙는데, 이게 얼마나 자주, 얼마나 길게 나는지 보고 싶다. Lambda는 실행이 끝날때마다 REPORT 로그를 찍는데, 콜드스타트면 거기 초기화 시간이 포함된다. 좋은건 이 값을 직접 정규식으로 뽑을 필요가 없다는 거다. CloudWatch가 `@initDuration` 이라는 필드로 알아서 추출해주고, 게다가 이 필드는 콜드스타트가 일어난 호출에만 붙는다. 그러니 이 필드가 있는 로그만 거르면 그게 곧 콜드스타트다.

~~~
filter @type = "REPORT" and ispresent(@initDuration)
| stats count() as coldStarts,
        avg(@initDuration) as avgInit,
        max(@initDuration) as maxInit
        by bin(1h)
~~~

`ispresent(@initDuration)` 으로 콜드스타트만 거르고, 한시간 단위(`bin(1h)`)로 묶어 횟수와 평균/최대 초기화 시간을 낸다. 이걸 보면 "새벽엔 트래픽이 없어서 콜드스타트가 몰린다" 같은 패턴이 한눈에 보인다.

사실 나도 처음엔 `Init Duration` 을 정규식으로 파싱했었다. 그러다 알게 된게, Lambda 로그의 주요 값들은 CloudWatch가 이미 `@` 붙은 필드로 다 뽑아둔다는 거다. `@duration`, `@billedDuration`, `@maxMemoryUsed`, `@memorySize`, `@requestId` 같은 것들이다. parse를 짜기 전에 이 자동 필드부터 확인하는게 먼저다. JSON으로 찍은 로그도 키를 알아서 필드로 잡아주니, 직접 파싱할 일이 생각보다 적다.

## **메모리를 과하게 잡고 있진 않나**
Lambda는 메모리를 얼마나 할당하냐에 따라 요금이 달라진다. 넉넉하게 잡아두면 마음은 편한데 돈이 샌다. 실제로 얼마나 쓰는지 보고 적정선을 찾고 싶다. Lambda는 실행이 끝날때마다 `REPORT` 라인을 찍는데, 거기 메모리 정보가 들어있다. 이건 parse도 필요없이 자동 필드로 잡힌다.

~~~
filter @type = "REPORT"
| stats max(@maxMemoryUsed / 1000 / 1000) as 최대사용MB,
        avg(@maxMemoryUsed / 1000 / 1000) as 평균사용MB,
        max(@memorySize / 1000 / 1000) as 할당MB
~~~

`@memorySize` 는 할당한 메모리, `@maxMemoryUsed` 는 실제로 쓴 최대치다. 할당은 512MB로 잡았는데 실제 최대 사용이 120MB 언저리라면, 메모리를 한참 과하게 잡고 있는거다. 줄이면 그만큼 비용이 빠진다. (물론 줄이면 CPU도 같이 줄어서 느려질수 있으니, 속도까지 같이 보고 정해야 한다)

## **요청 하나를 처음부터 끝까지 따라가기**
운영하다 보면 "이 요청이 어디서 꼬였나" 를 추적해야 할 때가 있다. 그래서 나는 발송이든 처리든, 요청마다 고유 id를 로그에 같이 찍게 해뒀다. 그럼 그 id로 필터를 걸어서 한 요청의 전체 흐름을 시간순으로 볼수 있다.

~~~
fields @timestamp, @message
| filter @message like /req_id=abc-123/
| sort @timestamp asc
~~~

이게 되려면 코드에서 로그를 찍을때 id를 항상 같이 박아주는 습관이 필요하다. 처음엔 귀찮은데, 장애 한번 추적해보면 이거 없이는 못 살겠다 싶어진다. 분산된 로그를 한 줄로 꿰는 열쇠가 이 id다.

## **알아둬야 할 함정 몇 개**
편하긴 한데 모르고 쓰면 당황하는 지점들이 있다.

첫째, `filter` 는 최대한 앞에 둬야 한다. Logs Insights는 쿼리가 스캔한 데이터 양으로 과금되고 속도도 거기서 갈린다. filter를 먼저 걸어서 데이터를 확 줄인 다음 parse나 stats를 하는것과, 반대로 하는것은 속도와 비용이 꽤 차이난다. 파이프라인 앞쪽에서 일찍 거를수록 좋다.

둘째, 결과가 최대 1만 건으로 잘린다. 집계(stats) 결과는 보통 1만건을 넘을 일이 없지만, 원본 로그를 그냥 쭉 뽑으려고 하면 1만건에서 짤린다. 더 봐야 하면 시간 범위를 좁히거나 필터를 더 걸어야 한다.

셋째, 비용. Logs Insights 쿼리 자체는 스캔한 1GB당 약 $0.005로 싼 편인데, 함정은 그 앞단이다. CloudWatch는 로그를 받아들이는(ingestion) 요금이 GB당 $0.50 수준으로 업계에서도 비싼 축이다. 로그를 무지성으로 다 때려박으면 쿼리비보다 적재비가 더 나온다. 그래서 안 볼 로그(디버그 레벨 같은)는 애초에 안 남기거나 보존기간을 짧게 두는게 중요하다. 그리고 자주 들여다보진 않지만 남겨는 둬야 하는 로그라면, Infrequent Access 라는 로그 클래스로 두면 적재비가 절반($0.25/GB)으로 준다. 쿼리 비용은 똑같이 나가니까, 실시간 알람 걸 일 없는 로그는 이쪽으로 빼두면 비용이 꽤 빠진다.

## **그럼 Athena 는 언제 쓰나**
예전에 [Athena로 ALB 로그를 분석하는 글]({% post_url 2024-03-23-aws-alb-log-athena %})을 쓴 적이 있다. 둘 다 로그를 쿼리하는 도구인데 쓰는 자리가 다르다.

Logs Insights는 "지금 막 난 장애" 를 보는 데 강하다. 셋업이 전혀 없고 콘솔에서 바로 쿼리가 되니까, 최근 며칠치 로그를 빠르게 디버깅할때 좋다. 반면 몇 달치를 모아서 추세를 보거나, 진짜 대용량을 다뤄야 하면 Athena가 맞다. CloudWatch 로그를 S3로 내보내서 Parquet으로 쌓아두고 Athena로 쿼리하면, 대량 구간에선 Logs Insights보다 훨씬(자료에 따라 10배 이상) 싸다.

참고로 "JOIN하려면 Athena" 라는 말도 이젠 반만 맞다. Logs Insights가 기존 자체 문법 말고 OpenSearch SQL이나 PPL 문법도 지원하게 되면서, 로그 그룹끼리 JOIN하는 정도는 Logs Insights에서도 된다. 다만 로그를 아예 다른 데이터셋이랑 엮거나, 대용량을 길게 훑는 작업은 여전히 Athena 쪽이 편하다.

정리하면 이렇다. 최근 로그를 빠르게 들여다보며 불을 끄는건 Logs Insights, 오래 쌓인 로그로 분석 리포트를 뽑는건 Athena. 나는 평소 디버깅은 Logs Insights로 하고, 장기 분석이 필요할때만 Athena로 넘어간다.

## **정리**
- Logs Insights 쿼리는 `filter | parse | stats` 파이프라인이다. 콘솔에서 로그 눈으로 뒤지지 말자.
- 콜드스타트(`@initDuration`), 메모리(`@maxMemoryUsed`) 같은 표준 값은 CloudWatch가 자동 필드로 뽑아준다. parse 짜기 전에 자동 필드부터 확인하자.
- filter는 앞에, 결과는 1만건 제한, 비용은 쿼리보다 적재(ingestion)가 더 무섭다.
- 최근 디버깅은 Logs Insights, 장기 대량 분석은 Athena로 나눠 쓴다.

콘솔에서 로그 스트림 하나하나 열어보던 시절을 생각하면, 이거 하나 익혀둔걸로 운영이 한결 수월해졌다. 거창한 모니터링 도구를 붙이기 전에, AWS가 기본으로 주는 이것부터 제대로 쓰는게 가성비가 좋다.
