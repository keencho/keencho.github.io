---
title: AWS ALB에 람다 없이 고정 IP로 접근하기
author: keencho
date: 2021-10-07 20:12:00 +0900
categories: [AWS]
tags: [AWS]
---

# **ALB에 고정 IP로 접근하기**  
AWS `ALB(Application Load Balancer)`는 `NLB(Network Load Balancer)`에 비해 가지는 이점들이 있습니다. 예를 들어 ALB는 서브도메인이나 path '/app/**' 같은 경로로 타겟 그룹에 요청을 전달할 수 있습니다. 
하지만 단점 하나가 있는데요, 바로 고정 IP를 부여하지 못한다는 것입니다. 서비스를 운영하다 보면 고객사가 내부 망을 사용하여 그들의 방화벽에 고정 IP를 등록해야 하기 때문에 고정 IP를 요구하는 경우가 있습니다.  

기존에는 람다, AWS GA 등을 사용하여 ALB에 고정 IP로 접근하곤 하였습니다. 하지만 2021년 9월 AWS에서 NLB ALB 로의 직접 트래픽 전달을 지원하게 됨으로써 더 쉽게 고정IP를 부여할수 있게 되었습니다. 
이 포스팅에서는 새로운 방법을 소개하도록 하겠습니다.  

## **기존 방법**
기존에 ALB에 고정 IP를 부여하는 방법으로는 람다, AWS Global Accelerator, Proxy EC2 Server 크게 3가지가 있는것 같습니다. 그중에서 제가 맡고있는 서비스에는 EC2에 nginx 만을 설치해 Proxy Server 처럼 사용하는 방식을 적용하였습니다. 
그리고 그 서버에는 아래와 같은 nginx config 파일을 작성 했었습니다.
```
server {
        listen 443 ssl;
        server_name api-proxy.homepick.com;

        ssl_certificate /etc/nginx/ssl/ssl.crt;
        ssl_certificate_key /etc/nginx/ssl/ssl.key;

        ssl_protocols SSLv2 SSLv3 TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;

        location / {
                access_log /var/log/nginx/proxy-api.log timed_combined;
                error_log /var/log/nginx/proxy-api-error.log;

                resolver 172.31.0.2 valid=10s;

                set $proxy "DNS NAME";

                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-FORWARDED-FOR $remote_addr;
                proxy_redirect off;
                proxy_read_timeout 120s;

                proxy_pass https://$proxy;
        }
}
```  

아시다시피 ALB는 동적 IP이기 때문에 도메인을 주면 안되고 DNS 주소 자체를 proxy_pass 가 바라볼수 있도록 세팅해야 합니다. 만약 www.xxx.com 형태의 도메인을 proxy_pass가 바라보게 설정한다면 ALB의 동적IP가 바뀌는 경우 nginx 캐시에 의해 올바른 엔드포인트를 찾아 들어가지 못해 클라이언트는 504 에러코드를 수신하게 되는 장애 상황이 발생하게 됩니다.  

이것뿐만이 아니고 이 Proxy Server 에서 들어오는 요청을 받는 웹서버 입장에서는 검증되지 않은 요청이기 때문에 받는 웹서버 nginx의 server 블록과 location 블록에 아래와 같은 설정을 추가하기도 했습니다. 
```
add_header 'Access-Control-Allow-Origin' 'domain';
add_header 'Access-Control-Allow-Credentials' 'true';
add_header 'Access-Control-Allow-Headers' 'Authorization,Accept,Origin,DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Content-Range,Range';
add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS,PUT,DELETE,PATCH';

...

if ($request_method = 'OPTIONS') {
    add_header 'Access-Control-Allow-Origin' 'domain';
    add_header 'Access-Control-Allow-Credentials' 'true';
    add_header 'Access-Control-Allow-Headers' 'Authorization,Accept,Origin,DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Content-Range,Range';
    add_header 'Access-Control-Allow-Methods' 'GET,POST,OPTIONS,PUT,DELETE,PATCH';
    add_header 'Access-Control-Max-Age' 1728000;
    add_header 'Content-Type' 'text/plain charset=UTF-8';
    add_header 'Content-Length' 0;

    return 204;
}
```  
이렇듯 고정 IP 설정을 위해 예외처리하는 설정이 많았고, 한번에 관리하기 힘든 문제, SSL 변경시 각기 적용 문제등 많은 문제가 있었지만 별다른 방법이 없었기 때문에 ~~눈물을 머금고~~ 이 상태로 서비스를 제공하고 있는 상태였습니다.  

## **NLB 에서 ALB 바로 바라보게 하기 (새로운 방법)**  
그러던 와중에 2021년 9월 27일에 AWS 사이트에 NLB에서 ALB로의 직접 트래픽 전달을 지원한다는 글이 올라오게 됩니다. [(링크)](https://aws.amazon.com/ko/about-aws/whats-new/2021/09/application-load-balancer-aws-privatelink-static-ip-addresses-network-load-balancer/) 설정하기가 쉽고 여러가지 이점이 있기 때문에 때문에 Proxy Server를 없애고 NLB가 ALB를 바라볼수 있게끔 설정하기로 결정하였습니다.  

로드밸런서로 ALB를 사용하고 있다는 전제하에 새로운 방식을 소개시켜 드리도록 하겠습니다. 만약 사용하고 계시지 않다면 [여기](https://keencho.github.io/posts/aws-cicd-4/)를 참고하셔서 ALB를 만들어 보세요.

### **타겟그룹 생성하기**
첫번째로 타겟그룹을 생성하겠습니다. AWS 콘솔에 접속하셔서 `Create target group`을 눌러주세요.
![1](/assets/img/custom/aws/alb-static-ip/1.PNG)

타겟그룹 생성화면에 들어왔다면 타겟 타입은 `Application Load Balancer` 를 선택하고 나머지 정보를 채워주세요. `Health check` 의 경우 기존 ALB가 바라보고 있는 대상 그룹의 health check 경로를 적어주시면 됩니다.
![2](/assets/img/custom/aws/alb-static-ip/2.PNG)
![3](/assets/img/custom/aws/alb-static-ip/3.PNG)  

다음단계로 넘어간다음 기존 ALB를 선택하고 `Create target group` 버튼을 클릭한다면 타겟그룹 생성은 끝입니다. 
![4](/assets/img/custom/aws/alb-static-ip/4.PNG)  

### **Network Load Balancer 생성하기**
다음은 NLB를 만들어보겠습니다. EC2 콘솔을 통해 로드밸런서 페이지에 들어오신후 `Load Balancer 생성` 버튼과 `Network Load Balancer` 버튼을 선택해 생성화면까지 들어와 주세요.  

이름을 입력하고 vpc를 선택한 후 가용영역을 선택합니다. 이때 AWS가 자동으로 할당하는 IP가 아닌 탄력적 IP(Elastic IP)를 사용하고 싶으시다면 탄력적 IP를 먼저 할당 받고 NLB를 생성해야 합니다.
![5](/assets/img/custom/aws/alb-static-ip/5.PNG)  

그 후 아까 만든 타겟 그룹을 `Foward to` 에서 선택하고 하단의 `Create load balancer`를 눌러 NLB를 생성합니다.
![6](/assets/img/custom/aws/alb-static-ip/6.PNG)  




