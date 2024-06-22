---
title: Terraform으로 AWS 무중단 배포 인프라 구성하기 - 1. 개요
author: keencho
date: 2023-12-09 08:12:00 +0900
categories: [AWS, Terraform]
tags: [AWS, ECS]
---

# **Terraform으로 AWS 무중단 배포 인프라 구성하기**
1. **개요**
2. [기초](/posts/terraform-aws-infra-2)
3. [네트워크](/posts/terraform-aws-infra-3)
4. [테스트 환경](/posts/terraform-aws-infra-4)
5. [운영환경 (프론트)](/posts/terraform-aws-infra-5)
6. [운영환경 (백엔드)](/posts/terraform-aws-infra-6)
7. [마무리](/posts/terraform-aws-infra-7)

# **Terraform으로 AWS 무중단 배포 인프라 구성하기 - 1. 개요**
최근 오픈하는 시스템을 ECS 기반 인프라로 변경하였다. 도입시 겪었던 시행착오나 기타 문제를 다시한번 정리하며 무중단 배포 인프라를 구성해본다.

AWS Web Console로도 인프라 구성이 가능히지만 코드를 사용해 인프라를 관리하기 위해 IaC중 하나인 [Terraform](https://www.terraform.io/) 을 사용한다. Terraform이 무엇인지는 자세하게 설명하진 않겠으나 필요시 간단하게 짚고 넘어가도록 한다.

[이 시리즈](https://keencho.github.io/posts/aws-cicd-1/) 와는 다르게 테스트 환경 구축, 운영 환경의 경우 ECS Fargate 사용, Terraform 사용, CloudFront 사용 등 실제 서비스에 적용할 수 있도록 구성해볼 것이다.

## **인프라 구성**
구성할 인프라를 간단하게 설명하면 다음과 같다. 도메인은 내가 보유중인 keencho.com 도메인을 사용한다.

1. Route 53은 다음 도메인의 트래픽을 각각 라우팅한다.
  - app-admin.keencho.com (관리자 - CloudFront)
  - app-user.keencho.com (사용자 - CloudFront)
  - app-admin-test.keencho.com (관리자 테스트 - Application Load Balancer)
  - app-user-test.keencho.com (사용자 테스트 - Application Load Balancer)
  - app-resources.keencho.com (리소스 - CloudFront)
2. CloudFront
  - 운영 트래픽, 리스소 트래픽을 담당
  - 운영 트래픽의 `/api/**` 경로로 시작하는 요청의 경우 Application Load Balancer로 전달한다.
  - `/api/**` 경로로 시작하지 않는 운영 요청은 react 앱이 배포되어 있는 s3 버킷으로 전송한다.
  - 리소스 트래픽은 리소스 s3 버킷으로 전송한다.
3. Application Load Balancer
  - 테스트 트래픽, 운영 api 트래픽 담당 (도메인에 따라 분기)
  - 테스트 트래픽은 2a 퍼블릭 서브넷에 위치하는 test + bastion 인스턴스로 전달
  - 운영 api 트래픽은 ECS Fargate 로 전송한다.
4. RDS - 운영 db는 RDS를 사용한다.
5. EFS - 어플리케이션 로그를 쌓기위한 용도로 EFS 를 사용한다.
6. ECR - 빌드한 도커  위이미지를 저장하기해 ECR을 사용한다.
7. CodeDeploy - 운영 앱 배포를 위해 CodeDeploy를 사용한다.

프론트단은 React, 백단은 SpringBoot를 사용하며 DB는 PostgreSQL을 사용한다.

## **인프라 다이어그램**
위 구성을 다이어그램으로 표현해보면 다음과 같다.

![서버구성도](/assets/img/custom/terraform-aws-infra/structure.png)

## **기타**
이 시리즈에서는 `Terraform`, `AWS CLI` 가 무엇인지 등은 설명하지 않는다.
```hcl
terraform apply
terraform plan
terraform destroy
```

위와 같은 명령어들도 마찬가지로 무엇인지는 설명하지 않으며 IaC, CLI 등에 기초 지식이 있다는 가정 하에 진행한다.

> :warning: Application Load Balancer에서 S3로 트래픽을 라우팅할수 있는 방법은 없다. 따라서 CloudFront에서 요청을 분기해야 하는데 이렇게 인프라를 구성하려면 Route 53 은 필수다. 아예 모든 리소스를 aws에서 구매하거나 본인이 소유한 도메인의 네임서버를 aws의 것으로 변경할 수 있어야 한다.



