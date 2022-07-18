---
title: AWS Application Load Balancer에 SSL 인증서 적용하기
author: keencho
date: 2022-07-09 20:12:00 +0900
categories: [AWS]
tags: [AWS, DevOps]
---

# **AWS Application Load Balancer에 SSL 인증서 적용하기**

## **개요**
이 포스팅에서는 .pem 확장자의 인증서를 AWS Application Load Balancer에 적용하는 방법에 대해 설명합니다. AWS ACM이 제공하는 인증서가 아닌 외부에서 ACM으로 가져온 인증서를 적용하는 방법에 대해 설명합니다.

## **1. AWS Certificate Manager 콘솔 이동**
AWS 콘솔 화면에서 Certificate Manager를 검색하여 인증서 관리 화면으로 이동하고, 좌측의 `인증서 가져오기` 버튼을 눌러 인증서를 가져오는 화면으로 이동합니다.

![alb-ssl-1](/assets/img/custom/aws/alb-ssl/alb-ssl-1.PNG)

## **2. 인증서 세부정보**
인증서 세부정보에 입력할 인증서 본문, 인증서 프라이빗 키, 인증서 체인이 필요합니다.

![alb-ssl-2](/assets/img/custom/aws/alb-ssl/alb-ssl-2.PNG)

제 경우 `DigiCert Global Root CA` 라는 발급자에게 발급받았기 때문에 `DigiCertCA` 이름의 pem 파일이 존재합니다.

일단 `인증서 프라이빗 키` 항목에 입력할 키를 만들어야 합니다. 키에 해당하는 파일을 열고 암호화를 해제해야 하는 파일인지 확인합니다. 아래 캡쳐와 같이 `Proc-Type` 항목과 `DEK-Info` 항목이 보인다면 암호화를 해체해야 하는 키파일 입니다.

![alb-ssl-3](/assets/img/custom/aws/alb-ssl/alb-ssl-3.PNG)

암호화를 해제한 새로운 .pem 파일을 만드는 방법은 간단합니다. 일단 openssl 이 설치된 환경이 필요한데요, Windows건 Linux건 상관 없습니다. 기존 key.pem 파일을 옮기고 아래 명령어를 수행합니다.
```
openssl rsa -in private-key.pem -out {생성할 파일 이름}
```

정상적인 파일이라면 비밀번호를 입력하라는 안내문구가 뜨는데요, 인증서를 발급받을 당시 발급자에게 제출한 비밀번호를 입력합니다. 파일이 생성되었다면 성공입니다.

```
인증서 본문 - cert.pem
인증서 프라이빗 키 - 암호화 해제한 키.pem
인증서 체인 - 발급기관명.pem
```

이중 `인증서 체인`의 경우 불필요한 경우도 있고 발급받은 인증서 형태에 따라 여러가지 단계로 조합해야 하는 경우도 있습니다. 자신의 인증서와 일치하는 인증서 체인 생성 방법은 검색을 통해 알아보시기 바랍니다.

## **3. 인증서 확인 & 생성**
모든 항목을 입력하였다면 2단계에서는 태그를 추가하고 3단계에서는 도메인, 만료 기간등을 확인한 후에 가져오기 버튼을 눌러 인증서를 가져옵니다.

![alb-ssl-5](/assets/img/custom/aws/alb-ssl/alb-ssl-5.PNG)

## **4. 인증서 적용 & 결과학인**
인증서를 가져왔으니 ALB에 적용해야 합니다. 이를 위해 로드 밸런서 콘솔 화면으로 이동하여 로드 밸런서를 선택하고 리스너 -> 인증서 보기/편집 순으로 이동합니다.

![alb-ssl-6](/assets/img/custom/aws/alb-ssl/alb-ssl-6.PNG)

저는 이미 인증서를 적용한 상태라 아래 캡쳐와 같이 인증서가 활성화된 상태이지만 처음 적용하는 경우 아무것도 없을 것입니다. `Add certificate` 버튼을 누르면 `Add certificate to listener` 화면이 등장하는데요, `ACM and IAM certificates` 부분에 방금 등록한 인증서가 표시될 것입니다. 해당 인증서를 체크하고 `Include as pending below` 버튼을 눌러 인증서를 적용하면 끝입니다.

![alb-ssl-7](/assets/img/custom/aws/alb-ssl/alb-ssl-7.PNG)

![alb-ssl-8](/assets/img/custom/aws/alb-ssl/alb-ssl-8.PNG)

이제 조금 시간을 두고 새로고침하며 내 사이트에 접속해보세요.

![alb-ssl-9](/assets/img/custom/aws/alb-ssl/alb-ssl-9.PNG)

위 캡쳐와 같이 브라우저에서 ssl 인증서를 확인할 수 있고 https 프로토콜로 접속할 수 있다면 성공입니다.


