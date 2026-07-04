# Day 2

## 날짜

2026년 5월 23일

## 오늘의 목표

AWS 정적 웹사이트 배포 프로젝트의 초기 아키텍처를 문서화한다.

## 완료한 작업

- 아키텍처 문서 작성
- AWS 정적 웹사이트 초기 아키텍처 흐름 정리
- 사용 예정 AWS 서비스 정리
- 보안 고려사항 정리
- 다음 AWS 실습 작업 항목 정리

## 초기 아키텍처

사용자 → Route 53 → CloudFront → S3 Bucket

## 사용 예정 서비스

### Amazon S3

정적 웹사이트 파일을 저장하는 서비스로 사용할 예정이다.

### Amazon CloudFront

사용자에게 정적 콘텐츠를 빠르게 전달하고 HTTPS 적용을 위해 사용할 예정이다.

### Amazon Route 53

도메인을 CloudFront와 연결하기 위해 사용할 예정이다.

### AWS Certificate Manager

HTTPS 인증서 발급과 관리를 위해 사용할 예정이다.

### IAM

AWS 리소스 접근 권한을 최소 권한 원칙에 따라 관리하기 위해 사용할 예정이다.

## 보안 고려사항

- S3 Bucket을 불필요하게 전체 공개하지 않는다.
- CloudFront를 통해 S3에 접근하도록 구성하는 방향을 고려한다.
- IAM 권한은 필요한 작업에 필요한 만큼만 부여한다.
- HTTPS 적용을 위해 AWS Certificate Manager 사용을 고려한다.

## 오늘 배운 점

- S3는 정적 파일 저장소로 사용할 수 있다.
- CloudFront는 콘텐츠 전송 속도와 보안을 개선할 수 있다.
- Route 53은 도메인 요청을 AWS 리소스로 연결할 수 있다.
- ACM은 HTTPS 적용에 필요한 인증서를 관리한다.
- IAM 권한은 필요한 만큼만 부여하는 것이 중요하다.

## 다음 작업

- AWS 콘솔에서 S3 버킷 생성
- 정적 웹사이트 호스팅 설정 검토
- `website/index.html` 파일 업로드
- S3 퍼블릭 접근 정책과 CloudFront 접근 방식 비교
