---
title: Application Load Balancer에 고정 IP 사용하기 (feat. VPC IPAM 통합)
author: keencho
date: 2025-04-12 20:12:00 +0900
categories: [AWS]
tags: [AWS]
---

# **Application Load Balancer에 고정 IP 사용하기 (feat. VPC IPAM 통합)**
aws 콘솔 보다가 이런걸 발견했다.
![2](/assets/img/custom/aws-alb-static-ip-2025/2.png)

궁금해서 찾아보니까 [이런 글](https://aws.amazon.com/ko/about-aws/whats-new/2025/03/application-load-balancer-integration-vpc-ipam/)을 확인할 수 있었다. (~~마참내!~~)
![1](/assets/img/custom/aws-alb-static-ip-2025/1.jpg)

ALB에 [고정IP를 사용하려고](https://keencho.github.io/posts/aws-alb-static-ip/) 앞에 NLB 붙이기, 프록시 서버 사용하기, Global Accelerator과 통합하기, Lambda 사용하기 등 다양한 방법들을 사용해왔다. 그런데 이제 ALB에 직접적으로 고정 IP를 사용할 수 있게 된 것이다.

## **VPC IPAM이란?**
Amazon VPC IP Address Manager(IPAM)는 AWS 워크로드의 IP 주소를 보다 쉽게 계획, 추적 및 모니터링할 수 있게 해주는 VPC 기능이다. IPAM을 사용하면 조직, 계정, VPC 및 AWS 리전 전체에서 IP 주소 배정을 중앙에서 관리하고 감사할 수 있다.
1. **범위(Scope)**: IPAM에서 가장 상위 컨테이너로, 단일 네트워크의 IP 공간을 나타낸다. IPAM 생성 시 퍼블릭 범위와 프라이빗 범위가 자동으로 생성된다.
2. **풀(Pool)**: 연속된 IP 주소 범위(CIDR)의 모음으로, 라우팅 및 보안 요구사항에 따라 IP 주소를 구성할 수 있다. 최상위 풀 내에 여러 개의 풀이 존재할 수 있다.
3. **할당(Allocation)**: IPAM 풀에서 AWS 리소스나 다른 IPAM 풀로 CIDR을 할당하는 것을 의미한다.

## **Application Load Balancer와 VPC IPAM의 통합**
2025년 3월에 AWS는 Application Load Balancer(ALB)와 Amazon VPC IPAM의 통합을 발표했다. 이 통합을 통해 고객은 로드 밸런서 노드에 IP 주소를 할당할 때 사용할 퍼블릭 IPv4 주소 풀을 직접 제공할 수 있게 되었다.

### **주요특징**
1. **IP 주소 풀 지정**: 고객은 퍼블릭 IPAM 풀을 구성할 수 있으며, 이 풀은 다음과 같은 주소로 구성될 수 있다.
  - 고객 소유 Bring Your Own IP Address(BYOIP)
  - Amazon이 제공하는 연속 IPv4 주소 블록
2. **비용 최적화**: BYOIP를 사용하여 퍼블릭 IPv4 비용을 최적화할 수 있다. 2024년 2월 1일부터 AWS에서는 모든 서비스 관리형 퍼블릭 IPv4 주소에 대해 요금을 부과하고 있다. 그러나 BYOIP 주소는 이러한 요금이 부과되지 않는다.
3. **간편한 허용 목록 관리**: Amazon이 제공하는 연속 IPv4 블록을 사용하여 사내 허용 목록 작성 및 운영 작업을 간소화할 수 있다.
4. **자동 전환 메커니즘:** IPAM 풀에서 공급되는 ALB의 IP 주소는 퍼블릭 IPAM 풀의 주소가 소진되면 AWS 관리형 IP 주소로 자동 전환된다. 이를 통해 규모 조정 이벤트 중에도 서비스 가용성을 최대화할 수 있다.

## **ALB와 IPAM 통합하기**
### **1. IPAM, 풀 생성**
AWS 콘솔에서 `Amazon VPC IP Address Manager` 를 검색해서 이동한 뒤, `IPAM` 과 `풀`을 생성한다.
![3](/assets/img/custom/aws-alb-static-ip-2025/3.png)
![4](/assets/img/custom/aws-alb-static-ip-2025/4.png)

CIDR 넷마스크 길이는 ALB의 가용 영역 갯수에 따라 선택하면 될 것 같다.

### **2. ALB IP 풀 편집**
사용중인 ALB -> 네트워크 매핑 -> IP 풀 편집을 클릭하여 IP 풀 편집 화면으로 이동한다.
![5](/assets/img/custom/aws-alb-static-ip-2025/5.png)

생성한 IPv4 IPAM 풀을 선택하고 저장하면 통합이 완료된다.

### **3. 확인**
다시 IPAM 콘솔로 돌아와 가용 영역의 갯수에 맞춰 IP가 리소스에 배정되었는지 확인해 보자.
![6](/assets/img/custom/aws-alb-static-ip-2025/6.png)

정확한 확인을 위해 nslookup 명령어로 할당된 IP가 리턴되는지 확인해 보자.
![8](/assets/img/custom/aws-alb-static-ip-2025/8.png)
![7](/assets/img/custom/aws-alb-static-ip-2025/7.JPG)

## **결론**
ALB와 IPAM의 통합을 통해 ALB에 고정 IP를 사용할 수 있게 되었다. 당연한 이야기겠지만 [이런 아키텍처](https://keencho.github.io/posts/aws-fargate-dynamic-operation/#%ED%98%84%EC%9E%AC-%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98)를 사용한다면 CloudFront에 고정 IP를 사용한것이 아니기 때문에 무용지물이다. (물론 이 경우 [Anycast 고정 IP](https://aws.amazon.com/ko/about-aws/whats-new/2024/11/amazon-cloudfront-anycast-static-ips/) 를 사용하면 된다.)

API 서비스 제공자라면 고객의 방화벽 설정을 위해 IPv4 주소를 요구하는 경우가 있다. `저희가 API를 사용하려는데, 방화벽 시스템 때문에 IP 등록이 필요합니다. DNS가 아닌 IPv4 주소를 알려주실 수 있을까요?` 와 같은 요청이 빈번하게 발생하기 때문이다. 이때 위 기능이 도움이 될 것이다.
