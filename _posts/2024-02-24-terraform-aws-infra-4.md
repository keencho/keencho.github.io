---
title: Terraform으로 AWS 무중단 배포 인프라 구성하기 - 4. 테스트 환경
author: keencho
date: 2024-02-24 08:12:00 +0900
categories: [AWS, Terraform]
tags: [AWS, ECS]
---

# **Terraform으로 AWS ECS 무중단 배포 인프라 구성하기 - 4. 테스트 환경**
이번 포스팅에서는 테스트 환경을 구성한다. 테스트 환경은 운영환경과는 다르게 백엔드, 프론트엔드, db가 모두 1개의 인스턴스에서 돌아가게 구성한다.

당연히 실제 비즈니스 서비스의 경우 실제 운영환경과 동일하게 환경을 구성해야 겠지만 이 시리즈에서는 그렇게까지 하진 않는다. ec2, key-pair등과 같은 리소스도 Terraform으로 생성해보기 위함이다.

테스트 환경의 트래픽은 `Route53 -> Application Load Balancer -> EC2` 로 전송된다.

테스트 환경은 무중단 배포를 고려하지 않는다.

## **리소스**
### **1. Application Load Balancer**
먼저 로드밸런서를 생성한다. 로드밸런서는 퍼블릭 서브넷을 가용영역으로 두어야 한다. 또한 80 포트와 443 포트를 개방하도록 하겠다.

```hcl
resource "aws_lb" "app-alb" {
  name = "app-alb"
  internal = false
  load_balancer_type = "application"
  security_groups = [aws_security_group.app-alb-sg.id]
  subnets = [for subnet in aws_subnet.public : subnet.id]

  enable_deletion_protection = true
}

resource "aws_security_group" "app-alb-sg" {
  name        = "app-alb-sg"
  description = "security group for application load balancer"
  vpc_id      = "${aws_vpc.vpc.id}"

  ingress {
    description = "http"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  ingress {
    description = "https"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "app-alb-sg"
  }
}
```

### **2. EC2**
다음으로 EC2를 생성한다. 나는 가난한 개발자 이므로 프리티어를 사용할 것이다.

#### **2.1 Key Pair**
키페어를 생성한다. [이 글](https://stackoverflow.com/a/49792833/13160032) 을 참고해 생성하였다. 다 만들었다면 output을 확인하여 파일을 생성해 추후 ssh 접속시 사용하면 된다.

```hcl
resource "tls_private_key" "pk" {
  algorithm = "RSA"
  rsa_bits  = 4096
}

resource "aws_key_pair" "app-test-key-pair" {
  key_name   = "app-test-key-pair"
  public_key = tls_private_key.pk.public_key_openssh
}

output "private_key" {
  value     = tls_private_key.pk.private_key_pem
  sensitive = true
}
```

#### **2.2 Security Group & Elastic IP**
다음으론 보안그룹과 고정 IP를 생성한다. 원래대로라면 개발자 pc의 ip만 ssh 접속 허용해야겠으나 편의상 모든 트래픽을 허용하도록 하겠다. 이 인스턴스로 들어오는 http 포트는 80 포트를 사용할 것이며 로드밸런서 에서 라우팅되는 요청만 허용할 것이다. 따라서 ingress 부분의 security groups 값을 로드밸런서의 보안그룹 id로 지정한다.

```hcl
resource "aws_security_group" "app-test-sg" {
  name        = "app-test-sg"
  description = "security group for app test instance"
  vpc_id      = "${aws_vpc.vpc.id}"

  ingress {
    description = "ssh"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "http"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    security_groups = [aws_security_group.app-alb-sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "app-test-sg"
  }
}

resource "aws_eip" "app-test-ec2-eip" {
  vpc   = true

  tags = {
    Name = "app-test-eip"
  }
}
```

#### **2.3 EC2**
준비가 되었으니 EC2를 생성한다. a 존에 있는 퍼블릭 서브넷에 생성될 것이며 프리티어로 사용이 가능한 t2.micro 타입을 선택했다. 기타 위 생성한 리소스들과 연결해준다.

```hcl
resource "aws_instance" "app-test-ec2" {
  ami = "ami-0b8414ae0d8d8b4cc"
  instance_type = "t2.micro"
  availability_zone = "${var.aws_region}${element(var.availability_zones, 0)}"
  subnet_id = aws_subnet.public[0].id
  key_name = aws_key_pair.app-test-key-pair.key_name

  vpc_security_group_ids = [aws_security_group.app-test-sg.id]

  tags = {
    Name = "app-test-ec2"
  }
}
```

인스턴스를 생성하였으니 위에서 생성한 eip와 연결해준다.

```hcl
resource "aws_eip" "app-test-ec2-eip" {
  vpc   = true
  instance = "${aws_instance.app-test-ec2.id}"

  tags = {
    Name = "app-test-eip"
  }
}
```

### **3. Application Load Balancer - Target Group & Routing**
테스트 EC2가 존재하는 대상 그룹을 생성하고 테스트 도메인을 해당 대상그룹으로 라우팅한다.

#### **3.1 Target Group**
타겟 그룹을 생성한다. 80포트로 트래픽을 전송받을 것이며 `/alb/health-check` 경로로 상태검사를 진행할 것이다.

```hcl
resource "aws_lb_target_group" "app-test" {
  name = "app-test"

  port = 80
  protocol = "HTTP"
  target_type = "instance"
  vpc_id = aws_vpc.vpc.id

  health_check {
    path = var.alb-health-check-path
    port = "traffic-port"
  }
}
```

#### **3.2 Listener**
로드밸런서 리스너를 생성한다. 80포트의 경우 443 포트로 리다이렉트 시킨다. 443 포트의 경우 앞서 AWS Certificate Manager 에서 생성한 ssl 인증서를 적용하고 대상그룹으로 트래픽을 포워딩한다.

```hcl
resource "aws_lb_listener" "app-alb-listener-http" {
  load_balancer_arn = aws_lb.app-alb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "redirect"

    redirect {
      port = "443"
      protocol = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

resource "aws_lb_listener" "app-alb-listener-https" {
  load_balancer_arn = aws_lb.app-alb.arn
  port              = "443"
  protocol          = "HTTPS"
  certificate_arn = aws_acm_certificate.ssl-certificate.arn

  default_action {
    type = "forward"
    target_group_arn = aws_lb_target_group.app-test.arn
  }
}
```

#### **3.3 Target Register**
테스트 인스턴스를 대상 인스턴스로 등록한다.

```hcl
resource "aws_lb_target_group_attachment" "app-test-attachment" {
  target_group_arn = aws_lb_target_group.app-test.id
  target_id        = aws_instance.app-test-ec2.id
  port             = 80
}
```

#### **3.4 Listener Rule**
로드밸런서 리스너 룰을 지정한다. 도메인이 `app-admin-test.keencho.com`, `app-user-test.keencho.com` 인 경우 테스트 인스턴스로 라우팅할 것이다.

```hcl
resource "aws_lb_listener_rule" "app-alb-test-rule" {
  listener_arn = aws_lb_listener.app-alb-listener-https.arn
  priority = 1

  action {
    type = "forward"
    target_group_arn = aws_lb_target_group.app-test.arn
  }

  condition {
    host_header {
      values = ["app-admin-test.keencho.com", "app-user-test.keencho.com"]
    }
  }
}
```

> :warning: 이 단계에서는 대상 그룹에 등록된 대상의 상태가 Unhealthy로 표시될 것입니다. 이는 인스턴스 내부에서 세팅해주지 않았기 때문이며 다음에 바로 설정해 보도록 하겠습니다.

### **4. Route 53 Record**
Route 53 에 레코드를 추가한다. 각 테스트 도메인으로 들어오는 트래픽이 alb로 향할수 있도록 설정할 것이다.

```hcl
resource "aws_route53_record" "app-admin-test" {
  zone_id = aws_route53_zone.keencho.id
  name    = "app-admin-test.keencho.com"
  type    = "A"

  alias {
    name                   = aws_lb.app-alb.dns_name
    zone_id                = aws_lb.app-alb.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "app-user-test" {
  zone_id = aws_route53_zone.keencho.id
  name    = "app-user-test.keencho.com"
  type    = "A"

  alias {
    name                   = aws_lb.app-alb.dns_name
    zone_id                = aws_lb.app-alb.zone_id
    evaluate_target_health = true
  }
}
```

테스트 환경을 위해 AWS에서 설정해 줘야할 것은 끝났다. 나머지는 테스트 인스턴스에 직접 접속해서 진행해야 한다.

### **5. EC2 설정**
이제 인스턴스에 직접 들어와 여러가지 설정을 해줘야 한다.

1. nginx 설치
2. java 설치
3. PostgreSQL 설치
4. 백엔드 앱 배포
5. 프론트 앱 배포

필자는 이러한 순서로 진행하였으며 이 포스팅은 인프라 구성에 대한 내용만 담을것이기 때문에 하나하나 어떻게 설치하고 배포하는지까지 작성하진 않는다.

사용한 스크립트는 [이곳](https://github.com/keencho/aws-infra-terraform-example/tree/master/script/test) 에서 확인할 수 있으며 꼭 필수로 해야할 작업 몇가지만 소개하도록 하겠다.

1. target group 상태검사 경로 지정 - nginx 설정 파일을 통해 `/alb/health-check` 경로로 접근한 경우 200 상태값을 리턴할 수 있도록 첫번째 서버 블록에 정의하였다.
2. `/api` 경로가 존재하는 요청과 아닌 경우 구분 - `/api` 경로가 없는 경우 프론트 리액트 앱으로, `/api` 경로가 있는 경우 백엔드 서버로 라우팅 될수 있도록 하였다.

이 단계까지 진행하였다면 대상그룹이 `Healthy` 상태로 변경되었는지 확인해보자. 변경되었다면 실제 도메인으로 접속해 기능이 정상적으로 동작하는지 확인해보면 된다.

![test admin](/assets/img/custom/terraform-aws-infra/test-complete1.png)

![test user](/assets/img/custom/terraform-aws-infra/test-complete2.png)


