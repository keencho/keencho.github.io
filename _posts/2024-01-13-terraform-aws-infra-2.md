---
title: Terraform으로 AWS 무중단 배포 인프라 구성하기 - 2. 기초
author: keencho
date: 2024-01-13 08:12:00 +0900
categories: [AWS, Terraform]
tags: [AWS, ECS]
---

# **Terraform으로 AWS 무중단 배포 인프라 구성하기**
1. [개요](/posts/terraform-aws-infra-1)
2. **기초**
3. [네트워크](/posts/terraform-aws-infra-3)
4. [테스트 환경](/posts/terraform-aws-infra-4)
5. [운영환경 (프론트)](/posts/terraform-aws-infra-5)
6. [운영환경 (백엔드)](/posts/terraform-aws-infra-6)
7. [마무리](/posts/terraform-aws-infra-7)

# **Terraform으로 AWS ECS 무중단 배포 인프라 구성하기 - 2. 기초**
앞서 설명했든 프론트단은 React, 백단은 Spring-Boot를 사용한다.

## **앱**
이 포스팅에서 앱을 구성하는 방법까지 포함하지는 않는다. [이 리포지토리](https://github.com/keencho/aws-infra-terraform-example) 에 필요한 코드는 모두 존재하니 필요한 분들은 참고하시기 바란다.

전체적인 구성 은 다음과 같다.
- 프론트 (React)
  - 모노레포 프로젝트 구성
- 백 (SpringBoot)
  - 멀티모듈 프로젝트 구성
  - 프론트 경로와 겹치지 않는 `contextPath` 필수 (이 예제에선 /api 사용)

프론트 경로와 겹치지 않는 `contextPath`가 필수인 이유는 CloudFront에서 경로에 따라 라우팅 정책을 다르게 가져가야 때문이다. (또한 CORS 문제 피하기) CloudFront는 운영 트래픽을 각각 s3와 Application Load Balancer 로 전송해야 하는데, 만약 contextPath가 `/app` 으로 모두 동일하다면 CloudFront가 경로에 따라 요청 분기를 하지 못하기 때문에 둘중 한곳에 유니크한 contextPath는 필수이다.

따라서 백단에 `/api` contextPath를 적용한다. (`/api` 경로가 존재하는 요청 -> `ALB`로 전송, 그외 -> `S3`로 전송)

만약 프론트 경로에 `/api/account/list`와 같은 경로가 존재하면 CloudFront는 요청을 ALB로 전송할 것이므로 의도하지 않은 동작이 될 것이다. 이를 염두하고 코드를 작성했다.

## **Terraform & AWS CLI**
- [이곳](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) 을 참고하여 `terraform`을 설치한다.
- [이곳](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) 을 참고하여 `AWS CLI`를 설치한다.
- [이곳](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-quickstart.html#getting-started-quickstart-new) 을 참고하여 `AWS CLI`를 설정한다.

모든 설정이 완료되었다면 아래 명령어를 통해 terraform 과 aws cli가 잘 구성되었는지 확인한다.

```
terraform
aws sts get-caller-identity
```

본격적인 AWS 리소스 구성은 다음 포스팅부터 진행한다.
