# SAA Review Notes

## S3

- 객체 스토리지 서비스
- 정적 웹사이트 호스팅 가능
- 버킷 정책, ACL, 퍼블릭 액세스 차단 설정 이해 필요
- 내구성 99.999999999%

## CloudFront

- CDN 서비스
- 엣지 로케이션을 통해 지연 시간 감소
- S3 오리진과 연결 가능
- HTTPS 적용 가능
- OAC 또는 OAI를 통해 S3 직접 접근 제한 가능

## Route 53

- AWS의 DNS 서비스
- 도메인 등록 및 레코드 관리 가능
- CloudFront와 Alias 레코드 연결 가능

## ACM

- SSL/TLS 인증서 발급 및 관리
- CloudFront에 적용하려면 us-east-1 리전에서 인증서 필요
