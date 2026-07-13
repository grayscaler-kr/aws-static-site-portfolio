# Progress

프로젝트 진행 과정과 주요 작업 내역을 날짜 순으로 기록합니다.

----------

## Day 1

**프로젝트 초기 구성**

### 작업 내용

-   GitHub 저장소 생성
-   기본 디렉토리 구조 생성
-   `index.html` 작성
-   GitHub Push
    

### 결과

-   프로젝트 기본 구조 구성 완료
    
----------

## Day 2

**아키텍처 설계 및 문서화**

### 작업 내용

-   AWS 정적 웹사이트 아키텍처 설계
-   사용 예정 AWS 서비스 역할 정리
  
### 결과

-   프로젝트 전체 구조 및 설계 방향 수립

----------

## Day 3

**S3 및 CloudFront 구축**

### 작업 내용

-   S3 Bucket 생성
-   정적 웹사이트 배포
-   CloudFront Distribution 생성
-   OAC(Origin Access Control) 적용
-   기본 서비스 동작 확인

### 결과

-   CloudFront를 통한 정적 웹사이트 정상 배포 확인

----------

## Day 4

**운영 기능 검증**

### 작업 내용

-   Custom Error Response 적용
-   `error.html` 반환 확인
-   Default Root Object 확인
-   CloudFront Cache 동작 확인
-   Cache Invalidation 테스트
-   S3 Versioning 확인

### 결과

-   캐시 정책 및 복구 기능 검증 완료

----------

## Day 5

**도메인 및 HTTPS 구성 준비**

### 작업 내용

-   Route 53 구조 학습
-   ACM 인증서 발급 과정 정리
-   사용자 지정 도메인 및 HTTPS 적용 절차 문서화

### 결과

-   Route 53 및 ACM 적용 준비 완료

----------

## Day 6

**도메인 등록 및 Route 53·ACM 구성**

### 작업 내용

- `grayscaler.dev` 도메인 등록
- Route 53 Public Hosted Zone 생성
- 도메인 등록 기관의 네임서버를 Route 53 NS로 변경
- `nslookup`을 통한 DNS 위임 확인
- `us-east-1` 리전에서 ACM 인증서 요청
- Route 53 CNAME 레코드를 이용한 DNS Validation

### 결과

- `grayscaler.dev`의 DNS 관리 권한을 Route 53으로 정상 위임
- `grayscaler.dev`, `*.grayscaler.dev` ACM 인증서 발급 완료

----------

## Day 7

**사용자 지정 도메인 및 HTTPS 연결**

### 작업 내용

- CloudFront에 `grayscaler.dev`, `www.grayscaler.dev`를 Alternate Domain Name으로 등록
- CloudFront에 ACM 인증서 연결
- Route 53에 CloudFront 배포를 대상으로 A/AAAA Alias 레코드 생성
- HTTP 요청의 HTTPS 리디렉션 확인
- DNS, HTTPS 및 인증서 적용 상태 검증
- `www.grayscaler.dev` 접속 실패 원인 분석 및 Alternate Domain Name 누락 해결

### 결과

- `grayscaler.dev`, `www.grayscaler.dev`를 통한 정적 웹사이트 HTTPS 접속 완료
- Route 53, CloudFront, ACM, S3 연동 및 HTTP→HTTPS 리디렉션 검증 완료

----------

## 상세 내용

[S3, CloudFront 연계](./s3-cloudfront-connect.md)

[CloudFront 에러 페이지 설정](./error-page.md)

[Route 53, ACM 및 HTTPS 구성](./route53-acm.md)


## 앞으로의 계획

-   GitHub Actions 기반 자동 배포
-   CloudFront Cache Invalidation 자동화
-   CloudWatch 모니터링
-   AWS Budgets 비용 관리
-   Terraform 기반 IaC 적용
