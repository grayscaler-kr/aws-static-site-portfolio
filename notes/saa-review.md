# SAA Review Notes

## S3

- 객체 스토리지 서비스
- 정적 웹사이트 호스팅 가능
- 버킷 정책, ACL, 퍼블릭 액세스 차단 설정 이해 필요
- 내구성 99.999999999%
- 정적 웹사이트 파일인 HTML, CSS, JavaScript, 이미지 파일 등을 저장할 수 있다.
- S3 정적 웹사이트 호스팅을 사용할 경우 웹사이트 엔드포인트가 생성된다.
- S3 버킷을 퍼블릭으로 열어 직접 접근하게 할 수도 있지만, 보안상 CloudFront를 앞단에 두는 구성이 더 권장된다.

## CloudFront

- CDN 서비스
- 엣지 로케이션을 통해 지연 시간 감소
- S3 오리진과 연결 가능
- HTTPS 적용 가능
- OAC 또는 OAI를 통해 S3 직접 접근 제한 가능
- 사용자는 S3에 직접 접근하지 않고 CloudFront를 통해 콘텐츠에 접근하도록 구성할 수 있다.
- CloudFront 캐시를 사용하면 반복 요청에 대한 응답 속도를 개선할 수 있다.
- 원본 파일이 변경되었는데 CloudFront에 이전 파일이 남아 있을 경우 캐시 무효화가 필요할 수 있다.

## Route 53

- AWS의 DNS 서비스
- 도메인 등록 및 레코드 관리 가능
- CloudFront와 Alias 레코드 연결 가능
- 사용자가 도메인 이름으로 접속하면 Route 53이 CloudFront 배포로 연결할 수 있다.
- AWS 리소스에 연결할 때 Alias 레코드를 사용할 수 있다.

## ACM

- SSL/TLS 인증서 발급 및 관리
- CloudFront에 적용하려면 us-east-1 리전에서 인증서 필요
- HTTPS 적용을 위해 CloudFront 배포에 ACM 인증서를 연결할 수 있다.
- 도메인 검증 방식으로 DNS 검증을 사용할 수 있다.

## IAM

- AWS 리소스에 대한 접근 권한을 관리하는 서비스
- 사용자, 그룹, 역할, 정책을 통해 권한을 제어한다.
- 필요한 권한만 부여하는 최소 권한 원칙을 따라야 한다.
- 포트폴리오 프로젝트에서도 불필요하게 관리자 권한을 사용하지 않는 것이 좋다.

## Security Notes

- S3 버킷을 불필요하게 전체 공개하지 않는다.
- 가능하면 CloudFront를 통해서만 S3 객체에 접근하도록 구성한다.
- S3 Public Access Block 설정과 Bucket Policy를 함께 검토해야 한다.
- HTTPS를 적용하여 사용자와 CloudFront 사이의 통신을 암호화한다.
- IAM 권한은 작업에 필요한 범위로 제한한다.

## Architecture Notes

- 기본 접속 흐름은 다음과 같다.

사용자 → Route 53 → CloudFront → S3 Bucket

- Route 53은 도메인 요청을 CloudFront로 연결한다.
- CloudFront는 S3를 오리진으로 사용해 정적 콘텐츠를 사용자에게 전달한다.
- S3는 정적 웹사이트 파일을 저장한다.
- ACM은 HTTPS 인증서를 제공한다.

