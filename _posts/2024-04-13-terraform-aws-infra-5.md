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

이 상태에서 버킷을 퍼블릭 액세스로 열어버리고 주소로 접근하면 웹 페이지가 뜬다. 그러나 앞단에 `CloudFront`를 두어 그곳에서 s3로 라우팅할 것이기 때문에 넘어간다.

### **2. CloudFront**
```terraform
resource "aws_cloudfront_origin_access_control" "admin-front" {
  name                              = "admin-front"
  description                       = "admin front"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

resource "aws_cloudfront_distribution" "admin-distribution" {
  origin {
    domain_name = aws_s3_bucket.app-prod-react.bucket_regional_domain_name
    origin_id   = aws_s3_bucket.app-prod-react.id
    origin_access_control_id = aws_cloudfront_origin_access_control.admin-front.id
    origin_path = "/admin"
  }

  enabled = true
  default_root_object = "index.html"
  comment = "admin distribution"

  aliases = ["app-admin.keencho.com"]

  default_cache_behavior {
    allowed_methods        = ["GET", "POST", "PUT", "OPTIONS", "DELETE", "PATCH", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = aws_s3_bucket.app-prod-react.id

    forwarded_values {
      query_string = false

      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl = 0
    default_ttl = 3600
    max_ttl = 86400
  }

  price_class = "PriceClass_100"

  restrictions {
    geo_restriction {
      restriction_type = "whitelist"
      locations        = ["KR"]
    }
  }

  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate.ssl-certificate-virginia.arn
    ssl_support_method = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  custom_error_response {
    error_code = 403
    error_caching_min_ttl = 10
    response_page_path = "/index.html"
    response_code = 200
  }
}
```

관리자 배포를 생성했다. 대체 도메인을 지정할 경우 SSL 인증서는 필수인데, 앞서 말했던 것처럼 여기엔 us-east-1 리전에 존재하는 인증서만 지정할 수 있다.

현 시점엔 라우팅되는 모든 트래픽이 s3 버킷의 `/admin` 폴더로 전송된다.

```terraform
resource "aws_route53_record" "app-admin" {
  zone_id = aws_route53_zone.keencho.id
  name    = "app-admin.keencho.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.admin-distribution.domain_name
    zone_id                = aws_cloudfront_distribution.admin-distribution.hosted_zone_id
    evaluate_target_health = true
  }
}
```

Route 53 레코드를 생성하여 `app-admin.keencho.com` 으로 들어오는 요청이 `CloudFront`로 라우팅 되도록 하였다. 사용자(user) 배포도 이름만 바꾸어 생성한다.

##### **S3 버킷 정책 변경**
현재 S3 버킷 정책은 아래 이미지와 같이 모든 퍼블릭 엑세스가 차단되어 있을 것이다.

![s3 bucket block access](/assets/img/custom/terraform-aws-infra/s3-bucket-block-access.png)

`CloudFront` 에서 온 요청은 허용하는 정책을 적용한다.

```terraform
resource "aws_s3_bucket_policy" "allow-from-cloudfront-policy" {
  bucket = aws_s3_bucket.app-prod-react.id

  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Sid      = "AllowCloudFrontServicePrincipal",
        Action   = "s3:GetObject",
        Effect   = "Allow",
        Resource = "${aws_s3_bucket.app-prod-react.arn}/*",
        Principal = {
          "Service": "cloudfront.amazonaws.com"
        },
        Condition = {
          StringEquals = {
            "AWS:SourceArn": [
              aws_cloudfront_distribution.admin-distribution.arn,
              aws_cloudfront_distribution.user-distribution.arn,
            ]
          }
        }
      }
    ]
  })
}
```

도메인으로 접속했을 때 의도한대로 페이지가 동작하는지 확인하자.

![prod front admin](/assets/img/custom/terraform-aws-infra/prod-front1.png)

![prod front user](/assets/img/custom/terraform-aws-infra/prod-front2.png)


