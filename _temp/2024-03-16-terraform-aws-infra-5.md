---
title: Terraform으로 AWS 무중단 배포 인프라 구성하기 - 5. 운영환경 (프론트)
author: keencho
date: 2024-03-16 08:12:00 +0900
categories: [AWS, Terraform]
tags: [AWS, ECS]
---

# **Terraform으로 AWS ECS 무중단 배포 인프라 구성하기 - 5. 운영환경 (프론트)**
이번 포스팅 부터는 운영 환경을 구성한다. 먼저 프론트에 해당하는 부분부터 구성해 보도록 한다.

순서는 `S3 버킷 생성 -> 앱 배포 -> CloudFront 배포 생성 -> CloudFront, S3 연결 -> Route53, CloudFront 연결`이 되겠다.

## **리소스**
### **1. S3 버킷**
먼저 S3 버킷을 생성한다.

