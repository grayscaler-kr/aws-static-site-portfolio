# AWS Static Website Hosting Portfolio

## 1. 프로젝트 개요

이 프로젝트는 AWS의 주요 서비스를 활용하여 정적 웹사이트를 배포하는 포트폴리오 프로젝트입니다.

정적 웹사이트 파일은 Amazon S3에 저장하고, Amazon CloudFront를 통해 사용자에게 콘텐츠를 전달하는 구조를 목표로 합니다.  
추후 Route 53을 이용해 도메인을 연결하고, AWS Certificate Manager를 통해 HTTPS 접속을 구성할 예정입니다.

또한 IAM 최소 권한 원칙, 보안 설정, 모니터링, 비용 관리 요소를 함께 학습하며 문서화하는 것을 목표로 합니다.

## 2. 핵심 사용 예정 AWS 서비스

- Amazon S3: 정적 웹사이트 파일 저장
- Amazon CloudFront: CDN 구성 및 HTTPS 제공
- Amazon Route 53: 도메인 연결
- AWS Certificate Manager: SSL/TLS 인증서 관리
- AWS IAM: 최소 권한 기반 접근 제어

## 3. 추가 확장 예정 AWS 서비스

- Amazon CloudWatch: 모니터링
- AWS CloudTrail: API 호출 기록 확인
- AWS Budgets: 비용 알림 설정

## 4. 목표 아키텍처

```text
사용자 → Route 53 → CloudFront → S3 Bucket
```

## 5. 프로젝트 목표

- S3 기반 정적 웹사이트 배포 구성
- CloudFront를 통한 콘텐츠 전송 및 HTTPS 적용
- Route 53을 이용한 사용자 지정 도메인 연결
- ACM을 이용한 SSL/TLS 인증서 적용
- IAM 최소 권한 원칙 적용
- S3 퍼블릭 접근 정책과 CloudFront 접근 방식 비교
- AWS SAA 학습 내용과 실습 내용을 연결하여 문서화
- 운영 관점에서 모니터링 및 비용 관리 요소 확장

## 6. 디렉토리 구조

```text
aws-static-site-portfolio/
├── architecture/
│   └── architecture.md
├── docs/
│   ├── day01.md
│   └── day02.md
│   └── day03.md
├── notes/
│   └── saa-review.md
├── website/
│   └── index.html
└── README.md
```

## 7. 진행 기록
- Day 1: 프로젝트 저장소 생성, 기본 디렉토리 구성, index.html 작성 및 GitHub 반영
- Day 2: AWS 정적 웹사이트 배포 아키텍처 문서화, 사용 예정 AWS 서비스 역할 정리, SAA 복습 노트 갱신
- Day 3: index.html 갱신 및 S3 배포, Cloudfront 연결, 사이트 열림 확인

## 8. 다음 작업
- AWS 콘솔에서 S3 버킷 생성
- 정적 웹사이트 호스팅 설정 검토
- website/index.html 파일 업로드
- S3 퍼블릭 접근 정책과 CloudFront 접근 방식 비교
