# 아키텍처 문서

## 1. 프로젝트 개요

이 프로젝트는 AWS 서비스를 활용하여 정적 웹사이트를 배포하는 포트폴리오 프로젝트입니다.

정적 웹사이트 파일은 Amazon S3에 저장하고, 사용자는 CloudFront를 통해 웹사이트에 접속하는 구조를 목표로 합니다.

추후 Route 53을 이용해 도메인을 연결하고, AWS Certificate Manager를 통해 HTTPS 접속을 구성할 예정입니다.

## 2. 초기 아키텍처

```text
사용자
  ↓
Route 53
  ↓
CloudFront Distribution + ACM Certificate
  ↓
S3 Bucket
```

## 3. 구성 요소

### Amazon S3

정적 웹사이트 파일을 저장하는 스토리지로 사용합니다.

### Amazon CloudFront

사용자에게 웹사이트 콘텐츠를 빠르게 전달하기 위한 CDN 서비스로 사용합니다.

### Amazon Route 53

도메인을 CloudFront와 연결하기 위한 DNS 서비스로 사용할 예정입니다.

### AWS Certificate Manager

HTTPS 적용을 위한 SSL/TLS 인증서 발급 및 관리에 사용할 예정입니다.
