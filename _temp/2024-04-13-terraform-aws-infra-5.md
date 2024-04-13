---
title: Terraform으로 AWS 무중단 배포 인프라 구성하기 - 5. 운영환경 (프론트)
author: keencho
date: 2024-04-13 08:12:00 +0900
categories: [AWS, Terraform]
tags: [AWS, ECS]
---

# **Terraform으로 AWS ECS 무중단 배포 인프라 구성하기 - 5. 운영환경 (프론트)**
이번 포스팅 부터는 운영 환경을 구성한다. 먼저 프론트에 해당하는 부분부터 구성해 보도록 한다.

순서는 `S3 버킷 생성 -> 앱 배포 -> CloudFront 배포 생성 -> CloudFront, S3 연결 -> Route53, CloudFront 연결`이 되겠다.

## **리소스**
### **1. S3 버킷**
먼저 S3 버킷을 생성한다.

```terraform
resource "aws_s3_bucket" "app-prod-react" {
  bucket = "app-prod-react"

  tags = {
    Name = "app-prod-react"
  }
}

resource "aws_s3_bucket_ownership_controls" "app-prod-react-ownership" {
  bucket = aws_s3_bucket.app-prod-react.id
  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_acl" "app-prod-react-acl" {
  depends_on = [aws_s3_bucket_ownership_controls.app-prod-react-ownership]

  bucket = aws_s3_bucket.app-prod-react.id
  acl    = "private"
}

resource "aws_s3_object" "app-prod-react-admin" {
  bucket = aws_s3_bucket.app-prod-react.id
  content_type = "application/x-directory"
  key = "admin/"
}

resource "aws_s3_object" "app-prod-react-user" {
  bucket = aws_s3_bucket.app-prod-react.id
  content_type = "application/x-directory"
  key = "user/"
}
```

버킷과 엑세스 제어 목록, 관리자와 유저 어플리케이션이 배포될 폴더를 정의했다.

관리자와 유저 어플리케이션을 업로드해 둘 것이다. 이 또한 terraform으로 진행이 가능하지만, 어짜피 앱 배포는 추후 `github action`을 통해 빌드하고 s3에 업로드할 것이므로 지금은 콘솔 환경에서 직접 업로드 한다.

![s3-upload](/assets/img/custom/terraform-aws-infra/s3-upload.png)

이 상태에서 버킷을 퍼블릭 액세스로 열어버리고 주소로 접근하면 웹 페이지가 뜬다. 그러나 앞단에 `Cloud Front`를 두어 그곳에서 s3로 라우팅할 것이기 때문에 이곳에선 넘어간다.

### **2. CloudFront**


