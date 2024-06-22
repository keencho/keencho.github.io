---
title: Terraform으로 AWS 무중단 배포 인프라 구성하기 - 7. 마무리
author: keencho
date: 2024-06-22 08:12:00 +0900
categories: [AWS, Terraform]
tags: [AWS, ECS]
---

# **Terraform으로 AWS 무중단 배포 인프라 구성하기**
1. [개요](/posts/terraform-aws-infra-1)
2. [기초](/posts/terraform-aws-infra-2)
3. [네트워크](/posts/terraform-aws-infra-3)
4. [테스트 환경](/posts/terraform-aws-infra-4)
5. [운영환경 (프론트)](/posts/terraform-aws-infra-5)
6. [운영환경 (백엔드)](/posts/terraform-aws-infra-6)
7. **마무리**

# **Terraform으로 AWS ECS 무중단 배포 인프라 구성하기 - 7. 마무리**
이 시리즈를 작성하면서 작성한 스크립트들이 완벽하다고 생각하진 않는다. 모듈화를 더 하거나 조금더 아름답게 스크립트들을 작성할 수 있었다고 생각한다.

또한 구성적으로도 미흡한 부분들이 존재한다.

- 엄격한 리소스 네이밍 규칙
- `Bastion Host` 분리
- `RDS` 프라이빗 서브넷으로 이전
- `ECR` 수명 주기 정책 규칙 설정
- 상세한 `Autoscaling` 규칙 정의
- 배포, Scaling 이벤트 발생시 AWS SNS 트리거
- 컨테이너 환경변수 안전하게 관리
- `Github Actions` 환경변수 안전하게 관리
- 그 외...

`Terraform` 사용 측면에서는 직접 콘솔로 하나하나 생성했을때나 AWS CLI, SDK 등으로 리소스를 생성할때보다 오류가 덜 발생하고 생산성 측면에서 많이 발전했다고 생각한다. 또한 비교적 읽기 쉽기 때문에 다른 사람과도 쉽게 공유할 수 있을것 같다.

무엇보다 `Terraform`의 가이드가 매우 잘되어 있는것 같다. 문제가 발생해도 가이드 문서를 참고하여 문제를 쉽게 해결할 수 있었다.

실제 서비스에 사용하려면 세세하게 옵션을 설정해야 하겠지만 이 글을 읽으신 분들은 모두 잘 적용할 수 있을것이라 생각한다.

> 모든 코드는 [이곳](https://github.com/keencho/aws-infra-terraform-example) 에서 확인할 수 있다.
