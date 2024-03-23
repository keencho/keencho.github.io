---
title: Athena로 ALB Access Log 분석하기
author: keencho
date: 2024-03-23 08:12:00 +0900
categories: [AWS]
tags: [AWS]
---

# **Athena로 ALB Access Log 분석하기**
`Appliation Load Balancer`에는 액세스 로그를 저장할 수 있는 기능이 있다. 이를 활성화 하면 S3에 모든 요청 로그를 저장하게 되는데, 동일한 날짜의 요청이라도 한 파일에 저장되는 것이 아니라 굉장히 많은 .gz 형식의 압축파일로 쪼개져 저장된다.

![img1](/assets/img/custom/aws-alb-log-athena/img1.png)*~~아름답다~~*

`/order` 라는 경로로 들어오는 요청을 검색해 본다고 가정해보자. 일단 파일을 찾기 위해 `마지막 수정` 필드를 확인해야 한다. 어찌저찌 파일을 찾았다면 파일을 다운받고 압축풀고 잘 읽어지지도 않는 파일을 찾아야 한다. 시간대가 좀 차이나는 요청을 비교해야 한다면 수많은 파일을 띄워놓고 하나하나 비교해야 한다.

AWS 에서 제공하는 [Amazone Athena](https://aws.amazon.com/ko/athena/?nc=sn&loc=0) 서비스를 이용하여 SQL문으로 원하는 데이터를 쉽게 추출할 수 있다.

Athena는 표준 SQL을 사용해 S3에 있는 데이터를 직접 간편하게 분석할 수 있는 대화형 쿼리 서비스이다. Athena는 서버리스이므로 인프라 설정이나 관리가 불필요하며, 실행하는 쿼리 또는 쿼리에 필요한 컴퓨팅을 기준으로 요금을 지불할 수 있다. 스캔한 데이터 1TB 당 $5의 요금이 부과된다. 예를들어 Athena를 사용해 100MB 의 데이터를 스캔하는 쿼리를 매일 100개씩 실행한다면 월별 $1.45의 요금이 부과된다.

![img2](/assets/img/custom/aws-alb-log-athena/img2.png)

엄청난 크기의 데이터를 다룬다면 비용이 부담되겠지만, 적당한 트래픽이 발생하는 서비스의 ALB 로그 분석용으로만 사용한다면 비용 압박은 거의 없다고 생각된다. (~~이런거 아끼는 회사는...ㅎ~~)

> 파티셔닝 관리를 위한 CREATE, ALTER 혹은 DROP TABLE 문과 같은 DDL문 또는 실패한 쿼리에는 요금이 부과되지 않습니다. 취소한 쿼리는 쿼리를 취소할 시점에 스캔된 총 데이터 양에 따라 요금이 청구됩니다.

## **Athena 설정**
쿼리를 날리기 앞서 필요한 설정들을 진행해 보자.

일단 쿼리 결과가 저장될 별도의 S3 버킷이 필요하다. 버킷을 생성하고 `Amazon Athena > 쿼리 편집기 > 설정 > 관리`로 이동한다.
![img3](/assets/img/custom/aws-alb-log-athena/img3.png)

`Location of query result` 항목에 방금 생성한 버킷을 지정한다.
![img4](/assets/img/custom/aws-alb-log-athena/img4.png)

## **데이터베이스 & 테이블 생성**
설정이 완료되었다면 편집기로 돌아와 데이터베이스를 생성한다.

```sql
CREATE DATABASE alb_log_db;
```

데이터베이스를 생성하였다면 좌측 `데이터 - 데이터베이스` 항목에서 방금 생성한 데이터베이스를 선택한다.
![img5](/assets/img/custom/aws-alb-log-athena/img5.png)

### **옵션1. 일반 테이블 생성**
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS alb_access_logs (
            type string,
            time string,
            elb string,
            client_ip string,
            client_port int,
            target_ip string,
            target_port int,
            request_processing_time double,
            target_processing_time double,
            response_processing_time double,
            elb_status_code int,
            target_status_code string,
            received_bytes bigint,
            sent_bytes bigint,
            request_verb string,
            request_url string,
            request_proto string,
            user_agent string,
            ssl_cipher string,
            ssl_protocol string,
            target_group_arn string,
            trace_id string,
            domain_name string,
            chosen_cert_arn string,
            matched_rule_priority string,
            request_creation_time string,
            actions_executed string,
            redirect_url string,
            lambda_error_reason string,
            target_port_list string,
            target_status_code_list string,
            classification string,
            classification_reason string,
            conn_trace_id string
            )
            ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
            WITH SERDEPROPERTIES (
            'serialization.format' = '1',
            'input.regex' =
        '([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*):([0-9]*) ([^ ]*)[:-]([0-9]*) ([-.0-9]*) ([-.0-9]*) ([-.0-9]*) (|[-0-9]*) (-|[-0-9]*) ([-0-9]*) ([-0-9]*) \"([^ ]*) (.*) (- |[^ ]*)\" \"([^\"]*)\" ([A-Z0-9-_]+) ([A-Za-z0-9.-]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^\"]*)\" ([-.0-9]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^ ]*)\" \"([^\s]+?)\" \"([^\s]+)\" \"([^ ]*)\" \"([^ ]*)\" ?([^ ]*)?( .*)?')
            LOCATION 's3://DOC-EXAMPLE-BUCKET/access-log-folder-path/'
```

위 SQL 문을 사용해 테이블을 생성한다. `LOCATION` 항목의 경로를 ALB 로그가 저장되어 있는 경로로 변경하면 된다.

### **옵션2. partition projection 기능을 활용해 테이블 생성 (추천)**
ALB 로그는 정해진 체계로 생성이 되기 때문에 `Athena partition projection` 기능을 활용해 쿼리 런타임을 줄이고 파티션 관리를 자동화 할수 있다. 파티션 프로젝션은 새로운 데이터를 추가할 때 자동으로 새로운 파티션을 추가하기 때문에 별도의 `ALTER TABLE ADD PARTITION`문을 사용하여 파티션을 수동으로 추가할 필요가 없다. 이를 통해 `Athena`를 최적화 (쿼리 실행시간 단축, 파티션 관리 자동화, 스캔 데이터 절약, 비용 절감) 할 수 있다.

`partition projection` 에 대해서는 [공식문서](https://docs.aws.amazon.com/athena/latest/ug/partition-projection.html) 를 확인하면 상세하게 알 수 있다.

ALB 로그 파일은 [설명](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-access-logs.html#access-log-file-format) 에 따르면 다음과 같은 형식으로 생성된다.

```
bucket[/prefix]/AWSLogs/aws-account-id/elasticloadbalancing/region/yyyy/mm/dd/aws-account-id_elasticloadbalancing_region_app.load-balancer-id_end-time_ip-address_random-string.log.gz
```

`yyyy/mm/dd` 형식으로 폴더가 생성됨을 확인할 수 있다. 따라서 `year`, `month`, `day`를 파티션 컬럼으로 지정하도록 하겠다.

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS sys_log (
            type string,
            time string,
            elb string,
            client_ip string,
            client_port int,
            target_ip string,
            target_port int,
            request_processing_time double,
            target_processing_time double,
            response_processing_time double,
            elb_status_code int,
            target_status_code string,
            received_bytes bigint,
            sent_bytes bigint,
            request_verb string,
            request_url string,
            request_proto string,
            user_agent string,
            ssl_cipher string,
            ssl_protocol string,
            target_group_arn string,
            trace_id string,
            domain_name string,
            chosen_cert_arn string,
            matched_rule_priority string,
            request_creation_time string,
            actions_executed string,
            redirect_url string,
            lambda_error_reason string,
            target_port_list string,
            target_status_code_list string,
            classification string,
            classification_reason string,
            conn_trace_id string
            )
            PARTITIONED BY
            ( year integer, month integer, day integer )
            ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
            WITH SERDEPROPERTIES (
            'serialization.format' = '1',
            'input.regex' =
        '([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*):([0-9]*) ([^ ]*)[:-]([0-9]*) ([-.0-9]*) ([-.0-9]*) ([-.0-9]*) (|[-0-9]*) (-|[-0-9]*) ([-0-9]*) ([-0-9]*) \"([^ ]*) (.*) (- |[^ ]*)\" \"([^\"]*)\" ([A-Z0-9-_]+) ([A-Za-z0-9.-]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^\"]*)\" ([-.0-9]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^ ]*)\" \"([^\s]+?)\" \"([^\s]+)\" \"([^ ]*)\" \"([^ ]*)\" ?([^ ]*)?( .*)?')
            LOCATION 's3://DOC-EXAMPLE-BUCKET/AWSLogs/<ACCOUNT-NUMBER>/elasticloadbalancing/<REGION>/'
            TBLPROPERTIES
            (
             "projection.enabled" = "true",

             "projection.year.type" = "integer",
             "projection.year.range" = "2024,2024",
             "projection.year.digits" = "4",

             "projection.month.type" = "integer",
             "projection.month.range" = "01,12",
             "projection.month.digits" = "2",

             "projection.day.type" = "integer",
             "projection.day.range" = "01,31",
             "projection.day.digits" = "2",

             "storage.location.template" = "s3://DOC-EXAMPLE-BUCKET/AWSLogs/<ACCOUNT-NUMBER>/elasticloadbalancing/<REGION>/${year}/${month}/${day}"
            )
```

`LOCATION` 항목과 `storage.locationon.template` 항목의 경로를 ALB 로그 버킷 경로로 변경하면 된다. 테이블을 생성하였다면 우측 테이블 섹션에서 `year`, `month`, `day` 컬럼을 찾아보자. 아래 캡쳐된 이미지와 같이 파티션된 컬럼이라고 표기되어 있을 것이다.

![img6](/assets/img/custom/aws-alb-log-athena/img6.png)

> 테이블은 생성되었는데 데이터가 조회되지 않는다면 [이 문서](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-access-logs.html#log-processing-tools)의 `Syntax` 문단을 확인해서 누락된 필드가 있는지 확인해 보세요. 예제의 경우 이 포스팅을 작성하는 시점의 필드들만을 포함하고 있으므로 추후 필드가 추가된다면 정상동작하지 않을 수 있습니다.

## **쿼리 실행**
테이블이 오류없이 생성되었다면 필요한 데이터를 분석하는 일만 남았다. 쿼리 작성후 `ctrl + enter` 로 쿼리를 실행할 수 있으며 하단 `쿼리 결과` 에서 결과를 확인할 수 있다.

![img7](/assets/img/custom/aws-alb-log-athena/img7.png)

## **예제**
날짜별 전체 요청 갯수 count
```sql
SELECT cast(year as VARCHAR) || cast(month as VARCHAR) || cast(day as VARCHAR) as date,
       count(*) as count
FROM sys_log
GROUP BY cast(year as VARCHAR) || cast(month as VARCHAR) || cast(day as VARCHAR)
```

2024년 3월 23일 11시 ~ 18시 30분 사이 요청중 처리 시간이 10초를 초과하는 요청을 처리시간 내림차순으로 정렬
```sql
SELECT *
FROM sys_log
WHERE
    (
    parse_datetime(time, 'yyyy-MM-dd''T''HH:mm:ss.SSSSSS''Z')
    BETWEEN
    parse_datetime('2024-03-23 02:00:00', 'yyyy-MM-dd HH:mm:ss')
    AND
    parse_datetime('2024-03-23 09:30:00', 'yyyy-MM-dd HH:mm:ss')
    )
AND
    target_processing_time > 10
ORDER BY target_processing_time DESC
```

3월 10일 요청중 네이버 웨일 브라우저에서 요청한 요청 갯수 count
```sql
SELECT count(*)
FROM sys_log
WHERE month = 03
  AND day = 10
  AND user_agent like '%Whale%'
```


