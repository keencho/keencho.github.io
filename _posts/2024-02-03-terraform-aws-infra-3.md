---
title: Terraform으로 AWS 무중단 배포 인프라 구성하기 - 3. 네트워크
author: keencho
date: 2024-02-03 08:12:00 +0900
categories: [AWS, Terraform]
tags: [AWS, ECS]
---

# **Terraform으로 AWS ECS 무중단 배포 인프라 구성하기 - 3. 네트워크**
이번 포스팅부터 본격적으로 AWS 리소스를 생성한다. 그 첫번째로 인프라 구성에 가장 기본이 되는 네트워크 관련 리소스(vpc, subnet 등...)부터 생성한다.

이 글에선 vpc, subnet등이 무엇인지 설명하진 않는다. [이 시리즈](https://keencho.github.io/posts/aws-cicd-1/) 를 따라가다 보면 개념들을 확인할 수 있으니 모르시는 분들은 읽어보시길 바란다.

시작하기전 기초 코드를 작성한다.

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.16"
    }
  }
  required_version = ">= 1.2.0"
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project = var.app_project_name
    }
  }
}

variable "app_name" {
  description = "The name of application"
  type = string
  default = "app"
}

variable "app_project_name" {
  description = "The name of Project"
  type = string
  default = "appProject"
}
```

전역적으로 `Project` 태그를 세팅하였다. 한 계정에 프로젝트가 1개이면 상관없지만, 프로젝트가 N개라면 프로젝트를 분리하여 지정했을 시 프로젝트 별로 비용을 확인할 수 있기 때문에 편리하다.

## **변수**
확장성(?) 을 위해 [변수](https://developer.hashicorp.com/terraform/language/values/variables)를 적극적으로 활용한다.

```hcl
variable "aws_region" {
  description = "The AWS region to deploy the VPC in"
  type        = string
  default     = "ap-northeast-2"
}

variable "availability_zones" {
  description = "List of availability zones to use"
  type        = list(string)
  default     = ["a", "b"]
}

variable "vpc_cidr" {
  description = "The CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "The CIDR blocks for the public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  description = "The CIDR blocks for the private subnets"
  type        = list(string)
  default     = ["10.0.3.0/24", "10.0.4.0/24"]
}

variable "db_subnet_cidrs" {
  description = "The CIDR blocks for the DB subnets"
  type        = list(string)
  default     = ["10.0.5.0/24", "10.0.6.0/24"]
}
```

위 변수들은 네트워크 리소스 생성에 필요한 변수들이다. 이 예제에서는 a, b 2개의 가용존을 사용한다. 만약 a, b, c, d 모두 사용하고 싶다면

```hcl
variable "availability_zones" {
  description = "List of availability zones to use"
  type        = list(string)
  default     = ["a", "b", "c", "d"]
}
```

위와 깉이 기본 값만 수정해주면 쉽게 가용존을 확장할 수 있다. (물론 서브넷 cidr 도 수정해 줘야 한다.) 웹 콘솔 환경에서 하나하나 클릭하면서 생성할때와 비교해 봤을때 생산성이 눈에 띄게 늘어남을 확인할 수 있을 것이다.

## **리소스**
### **1. VPC**
```hcl
resource "aws_vpc" "vpc" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.app_name}-vpc"
  }
}
```
첫번째로 VPC를 생성한다.

### **2. Internet Gateway**
```hcl
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "${var.app_name}-igw"
  }
}
```
다음으로 퍼블릭 서브넷에 붙일 용도로 Internet Gateway를 생성한다.

### **3. NAT Gateway**
```hcl
resource "aws_eip" "nat" {
  count = length(var.public_subnet_cidrs)
  vpc   = true

  tags = {
    Name = "${var.app_name}-nat-eip-${element(var.availability_zones, count.index)}"
  }
}

resource "aws_nat_gateway" "nat" {
  count         = length(var.public_subnet_cidrs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = element(aws_subnet.public[*].id, count.index)

  tags = {
    Name = "${var.app_name}-nat-${element(var.availability_zones, count.index)}"
  }

  depends_on = [aws_internet_gateway.igw]
}
```
다음으로 프라이빗 서브넷에 붙일 용도로 NAT Gateway를 생성한다. NAT Gateway는 탄력적 ip를 필요로 하기 때문에 이 또한 생성한다.

### **4. Routing Table**
```hcl
resource "aws_route_table" "rt-public" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "${var.app_name}-rt-public"
  }
}

resource "aws_main_route_table_association" "main" {
  vpc_id         = aws_vpc.vpc.id
  route_table_id = aws_route_table.rt-public.id
}

resource "aws_route" "route-public" {
  route_table_id         = aws_route_table.rt-public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

resource "aws_route_table" "rt-private" {
  count  = length(var.private_subnet_cidrs)
  vpc_id = aws_vpc.vpc.id

  tags = {
    Name = "${var.app_name}-rt-private-${element(var.availability_zones, count.index)}"
  }
}

resource "aws_route" "route-private" {
  count                = length(var.private_subnet_cidrs)
  route_table_id       = element(aws_route_table.rt-private[*].id, count.index)
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id       = element(aws_nat_gateway.nat[*].id, count.index)
}
```

다음으로 라우팅 테이블을 생성한다. 1개의 퍼블릭 라우팅 테이블과 각각의 존에 존재하는 2개의 프라이빗 라우팅 테이블을 생성하였으며 퍼블릭 라우팅 테이블을 기본 라우팅 테이블로 지정한다.

퍼블릭 서브넷의 아웃바운드 트래픽은 Internet Gateway로, 프라이빗 서브넷의 아웃바운드 트래픽은 NAT Gateway로 전송한다.

> :warning: VPC 생성시 AWS는 기본적으로 라우팅 테이블을 하나 생성한다. 위 구성에서 생성한 라우팅 테이블과는 전혀 관련없는 라우팅 테이블이다. 뭔가 깔끔(?) 하게 구성하려면 콘솔에서 혹은 CLI로 직접 해당 라우팅 테이블을 삭제하면 된다. (Terraform 차원에서는 살짝 어려운듯 하다.) 여기서는 진행하지 않는다.

### **5. Subnet**
```hcl
resource "aws_subnet" "public" {
  count                   = length(var.public_subnet_cidrs)
  vpc_id                  = aws_vpc.vpc.id
  cidr_block              = element(var.public_subnet_cidrs, count.index)
  map_public_ip_on_launch = true
  availability_zone       = "${var.aws_region}${element(var.availability_zones, count.index)}"

  tags = {
    Name = "${var.app_name}-subnet-public-${element(var.availability_zones, count.index)}"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.vpc.id
  cidr_block        = element(var.private_subnet_cidrs, count.index)
  availability_zone = "${var.aws_region}${element(var.availability_zones, count.index)}"

  tags = {
    Name = "${var.app_name}-subnet-private-${element(var.availability_zones, count.index)}"
  }
}

resource "aws_subnet" "private-db" {
  count             = length(var.db_subnet_cidrs)
  vpc_id            = aws_vpc.vpc.id
  cidr_block        = element(var.db_subnet_cidrs, count.index)
  availability_zone = "${var.aws_region}${element(var.availability_zones, count.index)}"

  tags = {
    Name = "${var.app_name}-subnet-db-private-${element(var.availability_zones, count.index)}"
  }
}
```
한개의 퍼블릭 서브넷과 4개의 프라이빗 서브넷 (2 for app, 2 for db)을 생성한다.

### **6. Subnet Association**
```hcl
resource "aws_route_table_association" "rt-public-association" {
  count          = length(var.public_subnet_cidrs)
  subnet_id      = element(aws_subnet.public[*].id, count.index)
  route_table_id = aws_route_table.rt-public.id
}

resource "aws_route_table_association" "rt-private-association" {
  count          = length(var.private_subnet_cidrs)
  subnet_id      = element(aws_subnet.private[*].id, count.index)
  route_table_id = element(aws_route_table.rt-private[*].id, count.index)
}

resource "aws_route_table_association" "rt-private-db-association" {
  count          = length(var.db_subnet_cidrs)
  subnet_id      = element(aws_subnet.private-db[*].id, count.index)
  route_table_id = element(aws_route_table.rt-private[*].id, count.index)
}
```
마지막으로 서브넷과 라우팅 테이블들을 명시적으로 연결한다.

## **리소스 맵**
![리소스 맵](/assets/img/custom/terraform-aws-infra/vpc-resource-map.png)

VPC 콘솔로 들어가 리소스 맵을 확인해 보자. 위와같이 리소스가 구성되어 있으면 된다. 더욱더 확실하게 확인하려면 각각의 리소스 콘솔로 이동하여 확인해보자.

## **Route 53 & SSL 인증서**
원할한 진행을 위해 Route 53 호스팅 영억을 생성하고 AWS Certificate Manager를 통해 SSL 인증서를 발급받아 두도록 하겠다. (추후 CloudFront 구성시 대체 도메인을 연결해야 하는데, 이때 SSL 인증서가 필수이다.)

각각의 record는 그때그때 추가하도록 한다.

```hcl
resource "aws_route53_zone" "keencho" {
  name = "keencho.com"

  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name = "app-keencho-route53"
  }
}

resource "aws_route53_record" "app-certificate-validation" {
  for_each = {
    for dvo in aws_acm_certificate.ssl-certificate.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = aws_route53_zone.keencho.id
}
```

```hcl
resource "aws_acm_certificate" "ssl-certificate" {
  domain_name       = "*.keencho.com"
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_acm_certificate_validation" "ssl-certificate-validation" {
  certificate_arn         = aws_acm_certificate.ssl-certificate.arn
  validation_record_fqdns = [for record in aws_route53_record.app-certificate-validation : record.fqdn]
}
```

AWS Certificate Manager에 SSL 인증서를 추가하고 Route 53 CNAME 레코드를 통해 손쉽게 검증할 수 있다.

주의할 점이 있다. 현 시점 생성된 SSL 인증서는 서울 리전 (ap-northeast-2)에 존재한다. 추후 CloudFront 생성시에는 버지니아 북부 (us-east-1) 에 존재하는 SSL 인증서만 사용할 수 있다. 따라서 us-ease-1 리전에도 인증서를 추가해 주도록 하겠다.

```hcl
provider "aws" {
  alias  = "us-east-1"
  region = "us-east-1"
}

resource "aws_acm_certificate" "ssl-certificate-virginia" {
  domain_name       = "*.keencho.com"
  validation_method = "DNS"
  provider = aws.us-east-1

  lifecycle {
    create_before_destroy = true
  }
}
```

어떤 로직이 숨겨져 있는지 모르겠지만 위 인증서도 앞서 진행한 `ssl-certificate-validation` 검증을 통해 검증받을 수 있다. (별도의 검증 리소스 블락이 필요하지 않음.) 따라서 생성후 조금만 기다리면 상태가 변경됨을 확인할 수 있다.

만약 도메인을 외부 도메인 업체 (가비아, 후이즈 등) 에서 구매하였다면 해당 업체 관리화면으로 이동해 네임서버를 AWS가 생성한 네임서버로 변경하도록 하자. 변경하지 않으면 무용지물이다.


