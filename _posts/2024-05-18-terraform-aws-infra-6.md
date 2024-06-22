---
title: Terraform으로 AWS 무중단 배포 인프라 구성하기 - 6. 운영환경 (백엔드)
author: keencho
date: 2024-05-18 08:12:00 +0900
categories: [AWS, Terraform]
tags: [AWS, ECS]
---

# **Terraform으로 AWS 무중단 배포 인프라 구성하기**
1. [개요](/posts/terraform-aws-infra-1)
2. [기초](/posts/terraform-aws-infra-2)
3. [네트워크](/posts/terraform-aws-infra-3)
4. [테스트 환경](/posts/terraform-aws-infra-4)
5. [운영환경 (프론트)](/posts/terraform-aws-infra-5)
6. **운영환경 (백엔드)**
7. [마무리](/posts/terraform-aws-infra-7)

# **Terraform으로 AWS ECS 무중단 배포 인프라 구성하기 - 6. 운영환경 (백엔드)**
백엔드 운영환경을 구축한다.

## **리소스**
### **1. RDS**
RDS를 생성한다.

```hcl
resource "aws_security_group" "app-rds-sg" {

  name        = "app-rds-sg"
  description = "security group for rds"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    description = "postgresSQL"
    from_port   = 5432
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "app-rds-sg"
  }
}

resource "aws_db_subnet_group" "app-rds-subnet-group" {
  name  = "app-rds-subnet-group"
  subnet_ids = aws_subnet.public[*].id

  tags = {
    name = "app-rds-subnet-group"
  }
}

resource "aws_db_instance" "app-rds" {
  db_name              = "app"
  identifier           = "app-rds"
  engine               = "postgres"
  engine_version       = "15.8"
  storage_type         = "gp2"
  allocated_storage    = 20
  instance_class       = "db.t3.micro"
  username             = var.db-username # secret
  password             = var.db-password # secret
  parameter_group_name = "default.postgres15"
  skip_final_snapshot  = true
  publicly_accessible  = true

  vpc_security_group_ids = [aws_security_group.app-rds-sg.id]
  db_subnet_group_name = aws_db_subnet_group.app-rds-subnet-group.name
}
```

rds가 생성되었다면 db 접속하여 유저, database 등 앱에 필요한 리소스들을 설정한다.

> :warning: 편의상 publicly_accessible, subnet group, security group을 퍼블릭으로 설정했다. 당연히 특정 bastion 어플리케이션환경 혹은 bastion host 에서만 접근 가능하게 해야한다.

### **2. EFS**
로그를 저장할 EFS 저장소를 생성한다. 2049 포트를 열어야 하며 네트워크상 프라이빗 서브넷에 위치하도록 하였다.

```hcl
resource "aws_security_group" "app-efs-sg" {
  name        = "app-efs-sg"
  description = "security group for ecs service"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    description = "allow 2049"
    from_port   = 2049
    to_port     = 2049
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "app-efs-sg"
  }
}

resource "aws_efs_file_system" "app-efs" {
  encrypted = true
  performance_mode = "generalPurpose"
  throughput_mode = "bursting"

  lifecycle_policy {
    transition_to_ia = "AFTER_90_DAYS"
  }
}

resource "aws_efs_mount_target" "app-efs-target" {
  count = "${length(aws_subnet.private.*.id)}"
  file_system_id  = "${aws_efs_file_system.app-efs.id}"
  subnet_id = "${element(aws_subnet.private.*.id, count.index)}"
  security_groups = ["${aws_security_group.app-efs-sg.id}"]
}
```

### **3. ECS**
ECS와 관련된 리소스들을 설정한다.

#### **1. ECR**
ECR을 생성한다.

```hcl
resource "aws_ecr_repository" "app-ecr" {
  name                 = "app-ecr"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = false
  }
}
```

원할한 테스트를 위해 이 단계에서 ECR 리포지토리에 이미지를 푸쉬해두자. 나의 경우 `nginx-latet`, `admin-latest`, `user-latest` 3개의 이미지를 푸쉬해 두었다.

#### **2. Task Definition**
```hcl
resource "aws_iam_role" "app-ecs-task-execution-role" {
  name = "ecsTaskExecutionRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Action = "sts:AssumeRole",
        Effect = "Allow",
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "app-ecs-task-execution-role-policy-attachment" {
  role       = aws_iam_role.app-ecs-task-execution-role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```

태스크를 정의하기전 필요한 리소스들이다. `AmazonECSTaskExecutionRolePolicy` 정책이 필요하기 떄문에 `app-ecs-task-execution-role`를 정의하여 붙여 두었다.

태스크 정의 설명전 잠시 서버 구성에 대해 이야기 하자면, 내가 구상한 구성은 각 `SpringBoot` 어플리케이션이 독립적인 컨테이너 형태로 존재하는것이 아닌 한 태스크 안에 `nginx`, `admin`, `user` 컨테이너가 존재하는 형태이다.

트래픽 흐름으로 설명하자면 `사용자 -> CloudFront -> ALB -> (태스크) -> Admin or User` (N개의 앱, N개의 ECS 서비스) 가 아닌 `사용자 -> CloudFront -> ALB -> (태스크) -> Nginx -> Admin or User` (N개의 앱, 1개의 ECS 서비스) 형태가 되는 것이다. 다른 시스템에는 (예를들어 회사에서 운영중인 서비스 등.) 독립적인 컨테이너 형태로 존재하는것이 더욱 안전하다고 생각한다.

물론 그에따른 리소스 비용 증가나 관리의 복잡성이 따라올 수 있으니 각 서비스별로 적당하게 서버 구성을 하면 될 것 같다. 일단 이 포스팅에선 `사용자 -> CloudFront -> ALB -> (태스크) -> Nginx -> Admin or User` 형태로 태스크를 정의한다.

```conf
user  nginx;
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 4096;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    server {
            listen 80;
            server_name app-admin.keencho.com
            client_max_body_size    1G;
            access_log off;

            location /health-check {
                return 200;
            }

            location /api/ {
                    proxy_redirect off;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header        Host $http_host;
                    proxy_connect_timeout 600;
                    proxy_send_timeout 600;
                    proxy_read_timeout 600;
                    send_timeout 600;
                    proxy_headers_hash_max_size 512;
                    proxy_headers_hash_bucket_size 128;

                    proxy_pass http://127.0.0.1:10000/api/;
            }
    }

    server {
            listen 80;
            server_name app-user.keencho.com
            client_max_body_size    1G;
            access_log off;

            location /api/ {
                    proxy_redirect off;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header        Host $http_host;
                    proxy_connect_timeout 600;
                    proxy_send_timeout 600;
                    proxy_read_timeout 600;
                    send_timeout 600;
                    proxy_headers_hash_max_size 512;
                    proxy_headers_hash_bucket_size 128;

                    proxy_pass http://127.0.0.1:10010/api/;
            }
    }

}
```

위와같은 nginx 설정파일을 사용한다.

```hcl
variable "nginx-container-name" {
  default = "nginx"
}

resource "aws_ecs_task_definition" "app-definition" {
  family = "app-definition"
  requires_compatibilities = ["FARGATE"]
  cpu = 1024
  memory = "2048"
  network_mode = "awsvpc"

  task_role_arn = aws_iam_role.app-ecs-task-execution-role.arn
  execution_role_arn = aws_iam_role.app-ecs-task-execution-role.arn

  runtime_platform {
    operating_system_family = "LINUX"
    cpu_architecture = "ARM64"
  }

  container_definitions = jsonencode([
    {
      name      = var.nginx-container-name
      image     = "${aws_ecr_repository.app-ecr.repository_url}:nginx-latest"
      essential = true
      portMappings = [
        {
          containerPort = 80
          hostPort      = 80
        }
      ],
      healthCheck = {
        command = ["CMD-SHELL", "curl -f http://localhost:80/health-check || exit 1"]
      }
    },
    {
      name      = "admin"
      image     = "${aws_ecr_repository.app-ecr.repository_url}:admin-latest"
      essential = true
      portMappings = [
        {
          containerPort = 10000
          hostPort      = 10000
        }
      ],
      environment = [
        {
          name  = "spring.profiles.active",
          value = "prod"
        },
        {
          "name": "logging.file.name",
          "value": "/app-efs/logs/prod/admin/$(curl -s $ECS_CONTAINER_METADATA_URI_V4/task | jq -r .TaskARN | cut -d '/' -f 3).log"
        },
        {
          "name": "server.port",
          "value": "10000"
        },
        {
          "name": "db_url",
          "value": "jdbc:postgresql://${aws_db_instance.app-rds.endpoint}/${aws_db_instance.app-rds.db_name}"
        },
        {
          "name": "db_username",
          "value": var.db-username
        },
        {
          "name": "db_password",
          "value": var.db-password
        },
      ],
      mountPoints: [
        {
          sourceVolume: "app-efs",
          containerPath: "/app-efs",
          readOnly: false
        }
      ],
      healthCheck = {
        command = ["CMD-SHELL", "curl -f http://localhost:10000/api/health-check || exit 1"]
      }
    },
    {
      name      = "user"
      image     = "${aws_ecr_repository.app-ecr.repository_url}:user-latest"
      essential = true
      portMappings = [
        {
          containerPort = 10010
          hostPort      = 10010
        }
      ],
      environment = [
        {
          name  = "spring.profiles.active",
          value = "prod"
        },
        {
          "name": "logging.file.name",
          "value": "/app-efs/logs/prod/user/$(curl -s $ECS_CONTAINER_METADATA_URI_V4/task | jq -r .TaskARN | cut -d '/' -f 3).log"
        },
        {
          "name": "server.port",
          "value": "10010"
        },
        {
          "name": "db_url",
          "value": "jdbc:postgresql://${aws_db_instance.app-rds.endpoint}/${aws_db_instance.app-rds.db_name}"
        },
        {
          "name": "db_username",
          "value": var.db-username
        },
        {
          "name": "db_password",
          "value": var.db-password
        },
      ],
      mountPoints: [
        {
          sourceVolume: "app-efs",
          containerPath: "/app-efs",
          readOnly: false
        }
      ],
      healthCheck = {
        command = ["CMD-SHELL", "curl -f http://localhost:10010/api/health-check || exit 1"]
      }
    }
  ])

  volume {
    name = "app-efs"

    efs_volume_configuration {
      file_system_id = aws_efs_file_system.app-efs.id
      root_directory = "/"
    }
  }

  lifecycle {
    ignore_changes = [container_definitions]
  }
}
```

- 시작유형: Fargate
- OS, 아키텍쳐: Linux/ARM64
- CPU: 2vCPU
- 메모리: 2 GB
- 볼륨: 앞에서 생성한 EFS 유형의 스토리지를 마운트, 루트 디렉토리는 `/`
- 컨테이너 정의
  1. nginx
     - 포트: 80
     - 상태확인: /health-check
  2. admin / user
     - 포트: 10000 / 10010
     - 상태확인: /api/health-check
     - EFS 볼륨 마운트
     - 환경변수
       - SpringBoot 프로필
       - 로깅파일 경로 지정: EFS 경로 + ... + 현재 컨테이너가 속한 태스크의 전체 Amazon 리소스 이름 (ARN)
       - db 설정값들 (여기서는 그냥 rds의 endpoint와 db_name을 끌어다 썼다. 실제 운영환경 이라면 S3에 환경 파일을 저장해두고 사용하는게 안전할것 같다.)

> :warning: ignore_changes 에 container_definitions 를 추가한다. 아무 수정 없어도 항상 terraform이 'force replacement' 를 시도하기 때문이다. 아마 'container_definitions'가 JSON 형식으로 정의되기 때문에 필드 순서나 미세한 차이로 인해 변경이 감지될 수 있어서 그런것 같다.

#### **2. Cluster & Service**
클러스터를 생성한다.

```hcl
resource "aws_ecs_cluster" "app-cluster" {
  name = "app-cluster"
}
```

서비스 생성전 로드밸런서와 연결할 타겟 그룹을 정의한다. 추후 CodeDeploy로 Blue / Green 배포를 진행할 것이므로 타겟 그룹을 2개 정의한다. 로드 밸런서에는 1번 대상그룹만 연결한다.

```hcl
resource "aws_lb_target_group" "app-ecs-service-tg1" {
  name = "app-ecs-service-tg1"

  port = 80
  protocol = "HTTP"
  target_type = "ip"
  vpc_id = aws_vpc.vpc.id

  health_check {
    path = var.alb-health-check-path
    port = "traffic-port"
  }
}

resource "aws_lb_target_group" "app-ecs-service-tg2" {
  name = "app-ecs-service-tg2"

  port = 80
  protocol = "HTTP"
  target_type = "ip"
  vpc_id = aws_vpc.vpc.id

  health_check {
    path = var.alb-health-check-path
    port = "traffic-port"
  }
}

resource "aws_lb_listener_rule" "app-alb-ecs-service-rule" {
  listener_arn = aws_lb_listener.app-alb-listener-https.arn
  priority = 2

  action {
    type = "forward"
    target_group_arn = aws_lb_target_group.app-ecs-service-tg1.arn
  }

  condition {
    host_header {
      values = ["app-admin.keencho.com", "app-user.keencho.com"]
    }
  }
}
```

다음으로는 서비스를 정의한다.

```hcl
resource "aws_security_group" "app-ecs-service-sg" {
  name        = "app-ecs-service-sg"
  description = "security group for ecs service"
  vpc_id      = aws_vpc.vpc.id

  ingress {
    description = "alb traffic"
    from_port   = 0
    to_port     = 65535
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
    Name = "app-ecs-service-sg"
  }
}

resource "aws_ecs_service" "app-ecs-service" {
  name = "app-ecs-service"
  cluster = aws_ecs_cluster.app-cluster.id
  task_definition = aws_ecs_task_definition.app-definition.arn
  desired_count = 1
  launch_type = "FARGATE"
  propagate_tags = "SERVICE"
  health_check_grace_period_seconds = 60

  network_configuration {
    subnets = aws_subnet.private[*].id
    security_groups = [aws_security_group.app-ecs-service-sg.id]
    assign_public_ip = false
  }

  deployment_controller {
    type = "CODE_DEPLOY"
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app-ecs-service-tg1.arn
    container_name = var.nginx-container-name
    container_port = 80
  }

  lifecycle {
    ignore_changes = [desired_count]
  }
}
```

- 시작유형: Fargate
- 태그전파: 서비스 기준
- 상태확인 유휴기간: 60초
- 배포 옵션: CodeDeploy
- 로드밸런서: 타겟그룹1에 등록, 80포트로 트래픽 라우팅
- 네트워크 구성: 프라이빗 서브넷에 존재, 로드밸런서 로부터 오는 트래픽만 허용, 나머지 금지

여기까지 오면 서비스와 태스크가 생성된다. 콘솔에서 상태를 확인해보자.

> :warning: 배포 옵션을 CodeDeploy로 지정했기 때문에 문제가 발생해도 서비스 업데이트를 할 수 없다. (현 시점에는 CodeDeploy와 연결되어 있지 않기 때문) 문제를 해결하려면 서비스를 지우고 다시 생성하는 수 밖에 없다.

#### **3. AutoScaling**
```hcl
resource "aws_iam_role" "app-ecs-autoscale" {
  name = "app-ecs-autoscale-iam-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Sid = "Autoscaling"
        Action = "sts:AssumeRole",
        Effect = "Allow",
        Principal = {
          Service = "application-autoscaling.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "app-ecs-autoscale" {
  role = aws_iam_role.app-ecs-autoscale.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceAutoscaleRole"
}

resource "aws_appautoscaling_target" "app-ecs-target" {
  min_capacity = 1
  max_capacity = 4
  resource_id = "service/${aws_ecs_cluster.app-cluster.name}/${aws_ecs_service.app-ecs-service.name}"
  role_arn = aws_iam_role.app-ecs-task-execution-role.arn
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace = "ecs"

  depends_on = [
    aws_ecs_service.app-ecs-service
  ]
}

resource "aws_appautoscaling_policy" "app-ecs-policy-scale-out" {
  name = "scale-out"
  policy_type = "StepScaling"
  resource_id = aws_appautoscaling_target.app-ecs-target.resource_id
  scalable_dimension = aws_appautoscaling_target.app-ecs-target.scalable_dimension
  service_namespace = aws_appautoscaling_target.app-ecs-target.service_namespace

  step_scaling_policy_configuration {
    adjustment_type = "PercentChangeInCapacity"
    cooldown = 1
    metric_aggregation_type = "Average"

    step_adjustment {
      metric_interval_lower_bound = 0
      scaling_adjustment = 100
    }
  }
}

resource "aws_cloudwatch_metric_alarm" "app-ecs-cpu-high" {
  alarm_name          = "app-ecs-cpu-high"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = "3"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = "60"
  statistic           = "Average"
  threshold           = "70"

  dimensions = {
    ClusterName = aws_ecs_cluster.app-cluster.name
    ServiceName = aws_ecs_service.app-ecs-service.name
  }

  alarm_actions = [aws_appautoscaling_policy.app-ecs-policy-scale-out.arn]
}
```

최소용량을 1, 최대용량을 4 로 지정하고 CPU에 따라 태스크를 `scale out` 하도록 구성하였다. 콘솔에서 확인해보자.

![autoscaling](/assets/img/custom/terraform-aws-infra/autoscaling.png)

### **4. CloudFront 수정**
앞서 운영환경 - 프론트를 구성할때 `CloudFront`로 들어오는 모든 트래픽은 S3로 전달되게 구성하였다. 맨 처음 개요에서 설명했듯 `/api/**` 경로로 시작하는 요청은 `Application Load Balancer`로 전달되도록 수정해야 한다.

```hcl
resource "aws_cloudfront_distribution" "admin-distribution" {
  origin {
    domain_name = aws_s3_bucket.app-prod-react.bucket_regional_domain_name
    origin_id   = aws_s3_bucket.app-prod-react.id
    origin_access_control_id = aws_cloudfront_origin_access_control.admin-front.id
    origin_path = "/admin"
  }

  # 추가
  origin {
    domain_name = aws_lb.app-alb.dns_name
    origin_id   = aws_lb.app-alb.id

    custom_origin_config {
      http_port                = 80
      https_port               = 443
      origin_protocol_policy   = "https-only"
      origin_ssl_protocols     = ["TLSv1.2"]
      origin_keepalive_timeout = 5
      origin_read_timeout      = 30
    }
  }

  enabled = true
  default_root_object = "index.html"
  comment = "admin distribution"

  aliases = ["app-admin.keencho.com"]

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = aws_s3_bucket.app-prod-react.id

    viewer_protocol_policy = "redirect-to-https"
    min_ttl = 0
    default_ttl = 3600
    max_ttl = 86400

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
  }

  # 추가
  ordered_cache_behavior {
    path_pattern = "/api/*"
    allowed_methods = ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
    cached_methods = ["GET", "HEAD"]
    target_origin_id       = aws_lb.app-alb.id

    viewer_protocol_policy = "redirect-to-https"
    default_ttl = 0
    max_ttl     = 0
    min_ttl     = 0

    forwarded_values {
      query_string = true
      headers      = ["*"]
      cookies {
        forward = "all"
      }
    }
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

글에는 관리자 수정본만 작성한다. `Application Load Balancer` 원본을 추가하였고 `/api/*` 경로 패턴은 로드밸런서로 라우팅 될 수 있도록 동작을 수정하였다.

### **5. CodeDeploy**
Blue / Green 배포를 위한 `CodeDeploy` 관련 리소스를 생성한다.

```hcl
resource "aws_codedeploy_app" "app-deploy-app" {
  compute_platform = "ECS"
  name = "app-deploy"
}

data "aws_iam_policy_document" "app-deploy-assume-role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["codedeploy.amazonaws.com"]
    }

    actions = ["sts:AssumeRole"]
  }
}

resource "aws_iam_role" "app-deploy-role" {
  name               = "app-deploy-role"
  assume_role_policy = data.aws_iam_policy_document.app-deploy-assume-role.json
}

resource "aws_iam_role_policy_attachment" "app-AWSCodeDeployRole" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole"
  role       = aws_iam_role.app-deploy-role.name
}

resource "aws_iam_role_policy_attachment" "app-AWSCodeDeployRoleForECS" {
  policy_arn = "arn:aws:iam::aws:policy/AWSCodeDeployRoleForECS"
  role       = aws_iam_role.app-deploy-role.name
}

resource "aws_codedeploy_deployment_group" "app-deploy-group" {
  app_name               = aws_codedeploy_app.app-deploy-app.name
  deployment_config_name = "CodeDeployDefault.ECSAllAtOnce"
  deployment_group_name  = "app-deploy-group"
  service_role_arn       = aws_iam_role.app-deploy-role.arn

  auto_rollback_configuration {
    enabled = false
  }

  blue_green_deployment_config {
    deployment_ready_option {
      action_on_timeout = "CONTINUE_DEPLOYMENT"
    }

    terminate_blue_instances_on_deployment_success {
      action                           = "TERMINATE"
      termination_wait_time_in_minutes = 1
    }
  }

  deployment_style {
    deployment_option = "WITH_TRAFFIC_CONTROL"
    deployment_type   = "BLUE_GREEN"
  }

  ecs_service {
    cluster_name = aws_ecs_cluster.app-cluster.name
    service_name = aws_ecs_service.app-ecs-service.name
  }

  load_balancer_info {
    target_group_pair_info {
      prod_traffic_route {
        listener_arns = [aws_lb_listener.app-alb-listener-https.arn]
      }

      target_group {
        name = aws_lb_target_group.app-ecs-service-tg1.name
      }

      target_group {
        name = aws_lb_target_group.app-ecs-service-tg2.name
      }
    }
  }
}
```

- 어플리케이션 생성
- CodeDeploy 관련 IAM Role 생성
- 배포그룹 생성
- 로드밸런서 지정 (Blue / Green 배포를 위해 앞에서 생성한 타겟그룹 1, 2 지정)

### **6. Github Actions 배포 스크립트 작성**
내가 사용한 Dockerfile이다.

```Dockerfile
FROM amazoncorretto:21

# 타임존 세팅
ENV TZ="Asia/Seoul"

ARG JAR_PATH

# 워크 디렉토리 세팅
WORKDIR /app-admin/

# 빌드된 jar 파일 복사
COPY $JAR_PATH /app-admin/app.jar

ENTRYPOINT ["java","-jar","app.jar"]
```

```Dockerfile
FROM nginx:latest
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
```

다음은 배포 스크립트다.

```yml
name: Deploy Spring Boot Application to ECS using CodeDeploy Blue / Green Deployment

on:
  workflow_dispatch:

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest

    steps:
      - name: Check out the repository
        uses: actions/checkout@v4

      - name: Setup JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'corretto'

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v3

      - name: Build Gradle
        working-directory: spring-boot
        run: |
          chmod +x ./gradlew
          ./gradlew bootjar --project-dir ./app-admin
          ./gradlew bootjar --project-dir ./app-user

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: {% raw %}${{ secrets.AWS_ACCESS_KEY_ID }}{% endraw %}
          aws-secret-access-key: {% raw %}${{ secrets.AWS_SECRET_ACCESS_KEY }}{% endraw %}
          aws-region: {% raw %}${{ secrets.AWS_REGION }}{% endraw %}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag and push image to Amazon ECR
        working-directory: spring-boot
        env:
          ECR_REGISTRY: {% raw %}${{ steps.login-ecr.outputs.registry }}{% endraw %}
        run: |
         docker build --build-arg JAR_PATH=app-admin/build/libs/*.jar --platform=linux/arm64 -t $ECR_REGISTRY/app-ecr:admin-latest -f app-admin/Dockerfile .
         docker build --build-arg JAR_PATH=app-user/build/libs/*.jar --platform=linux/arm64 -t $ECR_REGISTRY/app-ecr:user-latest -f app-user/Dockerfile .
         docker build --platform=linux/arm64 -t $ECR_REGISTRY/app-ecr:nginx-latest -f nginx.Dockerfile .

         docker push $ECR_REGISTRY/app-ecr:admin-latest
         docker push $ECR_REGISTRY/app-ecr:user-latest
         docker push $ECR_REGISTRY/app-ecr:nginx-latest

      - name: CodeDeploy Blue / Green Deployment
        working-directory: spring-boot
        run: |
          APPLICATION_NAME="{% raw %}${{ secrets.AWS_CODEDEPLOY_APPLICATION_NAME }}{% endraw %}"
          DEPLOYMENT_GROUP="{% raw %}${{ secrets.AWS_CODEDEPLOY_DEPLOYMENT_GROUP_NAME }}{% endraw %}"
          REGION="{% raw %}${{ secrets.AWS_REGION }}"
          REVISION_JSON='{
            "version": 1,
            "Resources": [
              {
                "TargetService": {
                  "Type": "AWS::ECS::Service",
                  "Properties": {
                    "TaskDefinition": "${{ secrets.AWS_CODEDEPLOY_TASK_DEFINITION }}{% endraw %}",
                    "LoadBalancerInfo": {
                      "ContainerName": "nginx",
                      "ContainerPort": 80
                    },
                    "PlatformVersion": "LATEST"
                  }
                }
              }
            ]
          }'

          # Create a new deployment
          aws deploy create-deployment \
            --cli-input-json "{\"applicationName\":\"$APPLICATION_NAME\",\"deploymentGroupName\":\"$DEPLOYMENT_GROUP\",\"revision\":{\"revisionType\":\"AppSpecContent\",\"appSpecContent\":{\"content\":\"$(echo $REVISION_JSON | sed 's/"/\\"/g')\"}}}" \
            --region $REGION
```

1. repository checkout
2. setup jdk21
3. setup gradle
4. build gradle (bootjar)
5. configure aws credentials
6. login ecr
7. build, tag and push image to ecr
   - OS, 아키텍쳐를 linux/arm64로 지정
8. CodeDeploy 배포 생성
   - 어플리케이션 지정
   - 배포 그룹 지정
   - 리전 지정
   - 태스크 정의 지정
   - 로드밸런서가 트래픽을 nginx(80 포트) 로 라우팅하도록 지정

위와같은 순서로 진행된다. 두근거리는 마음으로 `Run workflow` 버튼을 눌러보자.
