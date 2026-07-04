# AWS Static Website Hosting Portfolio

### 운영 관점에서 AWS 정적 웹사이트를 구축하고, 보안·캐시·장애 대응·비용 관리까지 단계적으로 확장하는 클라우드 운영 포트폴리오

AWS S3와 CloudFront를 이용해 정적 웹사이트를 배포하고,
운영 관점에서 보안, 캐시, 장애 대응, 비용 관리를 문서화한 포트폴리오입니다.

정적 콘텐츠 제공에 적합한 서버리스 아키텍처를 선택하여
운영 부담을 줄이고 비용 효율적인 서비스를 구성하는 것을 목표로 했습니다.

## 핵심 구성
- S3 Bucket Private 구성
- CloudFront OAC를 통한 S3 접근
- Custom Error Response 구성
- CloudFront Cache / Invalidation 테스트
- S3 Versioning 기반 복구 가능성 검토
- Route 53 / ACM 기반 도메인 및 HTTPS 적용 예정

## 목표 아키텍처
```text
사용자
   │
HTTPS
   │
Route 53
   │
CloudFront
   │
Origin Access Control (OAC)
   │
S3 Bucket (Private)
```

## 프로젝트 목표
- S3 Bucket Private 구성 통해 S3 직접 접근 차단
- CloudFront OAC 기반 보안 구성
- ACM 이용 통한 HTTPS 기반 안전한 서비스 제공
- Route 53 사용자 지정 도메인 연결
- CloudFront 캐시 동작 이해
- S3 Versioning을 통한 복구 검토
- IAM 최소 권한 구성
- AWS SAA 학습 내용을 실제 환경에서 검증
- 운영 관점의 모니터링 및 비용 관리 확장

## 사용한 AWS 서비스 및 선택 이유

|서비스|용도|선택이유|
|--|--|--|
|Amazon S3|정적 웹 콘텐츠 저장|서버 운영 없이 정적 콘텐츠를 저장하고 제공하기 위해 선택|
|Amazon CloudFront|콘텐츠 캐싱 및 HTTPS 제공|캐싱 통한 콘텐츠 전송 성능 향상, HTTPS 통한 안전한 서비스 제공 목적|
|AWS Route 53|사용자 지정 도메인(DNS) 연결|사용자 지정 도메인(DNS)을 관리하고 CloudFront와 연동하기 위해 사용|
|AWS Certificate Manager (ACM)|SSL/TLS 인증서 관리|HTTPS 적용을 위한 SSL/TLS 인증서를 무료로 발급 및 관리하기 위해 사용|
|IAM|접근 권한 관리|최소 권한 원칙(Principle of Least Privilege)을 적용하기 위해 사용|
|Origin Access Control (OAC)|S3 직접 접근 차단|CloudFront를 통해서만 S3에 접근하도록 구성하여 S3를 외부에 공개하지 않기 위해 사용|
|S3 Versioning|객체 버전 관리 및 복구|이전 버전의 객체를 보관하여 복구 가능성을 확보하기 위해 사용|

## 디렉토리 구조
```text
aws-static-site-portfolio/
├── architecture/
│   └── architecture.md
├── docs/
│   ├── progress.md
│   ├── s3-cloudfront-connect.md
│   ├── error-page.md
│   └── route53-acm-concept.md
├── website/
│   ├── index.html
│   └── error.html
└── README.md
```

## 운영 관점에서 고려한 내용

### 1. 보안
- S3 Bucket Public Access 차단
- CloudFront OAC를 통해서만 S3 접근 허용
- Bucket Policy는 `s3:GetObject`로 제한

#### 운영 고려 사항 
웹 콘텐츠는 CloudFront를 통해서만 접근하도록 구성하여  S3 직접 접근을 차단하였다.
이를 통해 S3 Bucket을 외부에 공개하지 않은 상태에서 콘텐츠 제공이 가능하도록 설계했다.

### 2. 캐시 운영
- CloudFront 캐시로 인해 S3 파일 수정 후에도 이전 파일이 보일 수 있음을 확인
- `/`와 `/index.html` 경로의 캐시 동작 차이 확인
- Invalidation을 통해 최신 파일 반영

#### 운영 고려 사항  
CloudFront 캐시 정책으로 인해 S3 파일을 수정해도 즉시 반영되지 않는 점을 확인하였다.
운영 중에는 Invalidation 또는 버전 기반 배포 전략을 고려해야 함을 확인하였다.

### 3. 장애 대응
- S3 직접 접근 시 403 발생 확인
- 존재하지 않는 객체 접근 시 Custom Error Response로 404 반환
- AccessDenied 발생 원인을 SSE-KMS 설정과 연결해 분석

#### 운영 고려 사항
장애 발생 시 아래 순으로 확인하였다.

1. CloudFront
2. OAC
3. Bucket Policy
4. S3 Object

### 4. 복구
- S3 Versioning을 활성화하여 이전 객체 버전으로 복구 가능한 구조를 확인

### 5. 비용 관리
- 서버를 운영하지 않는 정적 웹사이트 구조를 선택하여 운영 비용을 최소화하였다.
- 향후 AWS Budgets를 적용하여 예상 비용 초과 시 알림을 받을 수 있도록 확장할 예정이다.
- CloudFront와 Route 53의 과금 요소를 확인하고 운영 비용도 함께 고려하였다.

## 진행 기록
[progress.md 참고](./docs/progress.md)

## 향후 계획
- GitHub Actions를 이용한 자동 배포
- CloudFront Invalidation 자동화
- CloudWatch를 이용한 모니터링
- AWS Budgets를 이용한 비용 관리
- Terraform을 이용한 IaC 구성

## 이번 프로젝트에서 배운 점
이번 프로젝트를 통해
- S3 Public Access와 Bucket Policy의 차이를 이해하였다.
- CloudFront 캐시가 서비스 운영에 미치는 영향을 확인하였다.
- SSE-KMS 사용 시 CloudFront 접근 제한 문제가 발생할 수 있음을 경험하였다.
- 단순 구축보다 운영 중 발생 가능한 문제를 문서화하는 것이 중요함을 배웠다.

## 운영자로서 얻은 경험

이번 프로젝트는 단순히 정적 웹사이트를 배포하는 것보다, 실제 운영 과정에서 발생할 수 있는 문제를 직접 확인하고 해결하는 데 초점을 두었다.

특히 CloudFront 캐시, OAC를 이용한 보안 구성, S3 Versioning, SSE-KMS로 인한 접근 제한 문제 등을 직접 경험하면서 서비스 구축뿐 아니라 운영 과정에서 고려해야 할 요소들을 문서화하였다.

앞으로도 GitHub Actions를 이용한 자동 배포, CloudWatch 모니터링, AWS Budgets를 이용한 비용 관리, Terraform 기반 IaC 등을 단계적으로 적용하여 운영 자동화와 운영 효율성을 지속적으로 개선해 나갈 계획이다.
