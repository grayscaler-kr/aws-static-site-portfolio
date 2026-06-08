# Day 05 - Route 53 과 ACM 컨셉 리뷰

## 1. 학습 목표

Day 05에서는 사용자 지정 도메인과 HTTPS 적용을 위한 기본 개념을 정리한다.

이번 단계에서는 실제 도메인을 연결하기 전에 Route 53과 ACM의 역할을 먼저 이해하는 것을 목표로 한다.

- Route 53의 역할 이해
- Hosted Zone 개념 정리
- DNS Record 개념 정리
- Subdomain / Hostname 개념 정리
- CloudFront와 Route 53 연결 방식 이해
- ACM을 통한 SSL/TLS 인증서 개념 정리
- CloudFront에서 ACM 인증서를 사용할 때의 리전 규칙 이해

---

## 2. 전체 아키텍처 흐름

현재 프로젝트의 목표 아키텍처는 다음과 같다.

```text
User → Route 53 → CloudFront → S3 Bucket
```

각 서비스의 역할은 다음과 같다.

| Service | Role |
|---|---|
| Route 53 | 사용자가 입력한 도메인을 적절한 AWS 리소스로 연결하는 DNS 서비스 |
| CloudFront | S3의 정적 콘텐츠를 빠르고 안전하게 전달하는 CDN 서비스 |
| S3 | 정적 웹사이트 파일을 저장하는 스토리지 |
| ACM | HTTPS 적용을 위한 SSL/TLS 인증서를 발급 및 관리하는 서비스 |

Route 53은 사용자가 입력한 도메인이 어느 대상으로 연결되어야 하는지 알려준다.

예를 들어 사용자가 다음 주소로 접속한다고 가정한다.

```text
www.example.com
```

Route 53은 이 도메인을 CloudFront 배포 도메인으로 연결할 수 있다.

```text
www.example.com → dxxxxxxxxxxxxx.cloudfront.net
```

그 후 CloudFront는 S3 Bucket에서 정적 웹사이트 파일을 가져와 사용자에게 전달한다.

---

## 3. Route 53의 역할

Route 53은 AWS에서 제공하는 DNS 서비스이다.

DNS는 사람이 읽기 쉬운 도메인 이름을 실제 서버나 서비스 주소로 변환해주는 시스템이다.

예를 들어 사용자는 브라우저에 다음과 같이 입력한다.

```text
www.example.com
```

하지만 실제 인터넷 상에서는 이 요청이 특정 서버, CDN, 로드밸런서, API 서비스 등으로 연결되어야 한다.

Route 53은 이 도메인이 어디로 가야 하는지 알려주는 역할을 한다.

중요한 점은 Route 53을 사용한다고 해서 CloudFront가 반드시 필요한 것은 아니라는 것이다.

Route 53은 여러 대상으로 도메인을 연결할 수 있다.

```text
User → Route 53 → CloudFront
```

```text
User → Route 53 → S3 Static Website Endpoint
```

```text
User → Route 53 → EC2
```

```text
User → Route 53 → Elastic Load Balancer
```

```text
User → Route 53 → API Gateway
```

```text
User → Route 53 → External Hosting Service
```

즉, Route 53은 도메인 연결 담당이고, CloudFront는 연결 대상 중 하나이다.

다만 이번 프로젝트에서는 정적 웹사이트를 더 안전하고 빠르게 제공하기 위해 다음 구조를 사용한다.

```text
User → Route 53 → CloudFront → S3 Bucket
```

이 구조를 사용하는 이유는 다음과 같다.

- CloudFront를 통해 HTTPS 적용이 쉽다.
- S3 Bucket을 직접 공개하지 않고 CloudFront를 통해 접근하도록 구성할 수 있다.
- CDN 캐시를 통해 콘텐츠를 빠르게 전달할 수 있다.
- Custom Error Response, Default Root Object, Cache Behavior 등을 활용할 수 있다.

---

## 4. Hosted Zone 개념

Hosted Zone은 특정 도메인의 DNS 레코드를 관리하는 공간이다.

예를 들어 내가 다음 도메인을 가지고 있다고 가정한다.

```text
example.com
```

Route 53에서는 이 도메인을 위한 Hosted Zone을 만들 수 있다.

```text
Hosted Zone: example.com
```

그리고 이 Hosted Zone 안에 여러 DNS 레코드를 추가할 수 있다.

예를 들면 다음과 같다.

```text
example.com
```

```text
www.example.com
```

```text
api.example.com
```

```text
mail.example.com
```

```text
blog.example.com
```

```text
admin.example.com
```

Hosted Zone은 쉽게 말하면 `example.com`이라는 도메인 주소 체계 전체를 관리하는 공간이다.

---

## 5. Subdomain / Hostname 개념

Hosted Zone 안에서는 `www`, `api`, `mail`, `blog` 같은 이름을 만들고 관리할 수 있다.

예를 들어 Hosted Zone이 다음과 같다면:

```text
example.com
```

아래와 같은 하위 도메인을 만들 수 있다.

```text
www.example.com
```

```text
api.example.com
```

```text
mail.example.com
```

```text
admin.example.com
```

```text
blog.example.com
```

```text
dev.example.com
```

```text
test.example.com
```

다만 정확히 말하면 Route 53에서 하위 도메인 자체를 생성한다기보다는, 해당 이름에 대한 DNS 레코드를 만드는 것이다.

예를 들어:

```text
api.example.com
```

이라는 이름을 Route 53에 추가한다고 해서 실제 API 서버가 자동으로 생성되는 것은 아니다.

Route 53에는 다음과 같은 연결 정보를 등록하는 것이다.

```text
api.example.com → API Gateway 또는 Load Balancer
```

즉, Route 53은 이름표를 만들고 그 이름표가 어디를 가리키는지 알려준다.

예시는 다음과 같다.

| Name | Full Domain | Target |
|---|---|---|
| empty | example.com | CloudFront |
| www | www.example.com | CloudFront |
| api | api.example.com | API Gateway |
| mail | mail.example.com | Mail Server |
| admin | admin.example.com | Load Balancer |
| blog | blog.example.com | External Blog Service |

여기서 `example.com`처럼 앞에 별도 이름이 없는 도메인을 Root Domain, Apex Domain 또는 Naked Domain이라고 부른다.

---

## 6. DNS Record 개념

DNS Record는 특정 도메인 이름이 어떤 대상을 가리키는지 정의하는 설정이다.

Route 53 Hosted Zone 안에는 여러 종류의 DNS Record를 만들 수 있다.

대표적인 레코드는 다음과 같다.

| Record Type | Description |
|---|---|
| A Record | 도메인을 IPv4 주소로 연결 |
| AAAA Record | 도메인을 IPv6 주소로 연결 |
| CNAME Record | 도메인을 다른 도메인 이름으로 연결 |
| MX Record | 메일 서버 정보 지정 |
| TXT Record | 도메인 검증, SPF, DKIM 등 텍스트 정보 저장 |
| NS Record | 해당 도메인을 관리하는 네임서버 정보 |
| SOA Record | 도메인 권한 및 기본 DNS 정보 |

예를 들어 다음과 같이 사용할 수 있다.

```text
www.example.com → CloudFront
```

```text
api.example.com → API Gateway
```

```text
mail.example.com → Mail Server
```

DNS Record는 도메인 이름과 실제 연결 대상 사이의 매핑 정보라고 볼 수 있다.

---

## 7. Alias Record 개념

AWS Route 53에는 일반 DNS 레코드 외에 Alias Record라는 기능이 있다.

Alias Record는 AWS 리소스를 대상으로 도메인을 연결할 때 유용하다.

예를 들어 CloudFront 배포 도메인이 다음과 같다고 가정한다.

```text
dxxxxxxxxxxxxx.cloudfront.net
```

Route 53에서 다음과 같이 연결할 수 있다.

```text
www.example.com → dxxxxxxxxxxxxx.cloudfront.net
```

이때 CloudFront 같은 AWS 리소스에 연결할 때는 Alias Record를 사용할 수 있다.

Alias Record의 장점은 다음과 같다.

- CloudFront, ELB, S3 Website Endpoint 등 AWS 리소스와 연결하기 좋다.
- Root Domain에도 사용할 수 있다.
- AWS 리소스의 IP 변경을 직접 관리하지 않아도 된다.

특히 Root Domain에는 일반 CNAME을 사용할 수 없다.

```text
example.com
```

이런 Root Domain을 CloudFront에 연결하려면 Route 53의 Alias Record를 사용하는 것이 일반적이다.

예시는 다음과 같다.

```text
example.com → CloudFront Distribution
```

```text
www.example.com → CloudFront Distribution
```

---

## 8. Wildcard Record 개념

Route 53에서는 Wildcard Record도 만들 수 있다.

예를 들어 다음과 같은 레코드를 만들 수 있다.

```text
*.example.com
```

이렇게 설정하면 아래와 같은 여러 하위 도메인이 같은 대상으로 연결될 수 있다.

```text
a.example.com
```

```text
test.example.com
```

```text
dev.example.com
```

```text
anything.example.com
```

Wildcard Record는 많은 하위 도메인을 한 번에 처리할 때 유용하다.

하지만 운영 환경에서는 주의가 필요하다.

원하지 않는 하위 도메인까지 모두 같은 대상으로 연결될 수 있기 때문이다.

---

## 9. Route 53 비용 관점

Route 53에서 여러 DNS 레코드를 만든다고 해서 레코드 개수만큼 비용이 크게 증가하는 구조는 아니다.

비용에 주로 영향을 주는 요소는 다음과 같다.

- Hosted Zone 개수
- DNS Query 수
- 도메인 등록 비용
- Health Check 사용 여부

즉, `www`, `api`, `blog`, `admin` 같은 레코드를 몇 개 더 추가하는 정도는 일반적인 포트폴리오 프로젝트에서는 큰 부담이 되지 않는다.

다만 실제 서비스에서 트래픽이 많아지고 DNS Query가 많아지면 비용이 증가할 수 있다.

---

## 10. ACM의 역할

ACM은 AWS Certificate Manager의 약자이다.

ACM은 HTTPS 적용에 필요한 SSL/TLS 인증서를 발급하고 관리하는 서비스이다.

사용자가 다음 주소로 안전하게 접속하려면 HTTPS 인증서가 필요하다.

```text
https://www.example.com
```

HTTPS를 사용하면 브라우저와 서버 사이의 통신이 암호화된다.

ACM을 사용하면 AWS 서비스와 연동할 수 있는 인증서를 쉽게 발급받고 관리할 수 있다.

이번 프로젝트에서는 CloudFront에 ACM 인증서를 연결하여 사용자 지정 도메인에 HTTPS를 적용할 예정이다.

```text
User → https://www.example.com → Route 53 → CloudFront → S3 Bucket
```

---

## 11. ACM 인증서 검증 방식

ACM에서 인증서를 발급받으려면 해당 도메인을 실제로 소유하고 있다는 것을 증명해야 한다.

대표적인 검증 방식은 다음과 같다.

| Validation Method | Description |
|---|---|
| DNS Validation | DNS 레코드를 추가하여 도메인 소유권 검증 |
| Email Validation | 도메인 관리자 이메일로 검증 |

일반적으로 AWS에서는 DNS Validation 방식을 많이 사용한다.

Route 53을 사용 중이라면 ACM에서 제공하는 검증용 CNAME 레코드를 Hosted Zone에 추가하여 도메인 소유권을 검증할 수 있다.

예시는 다음과 같다.

```text
_acme-challenge.example.com → validation-value.acm-validations.aws
```

검증이 완료되면 ACM 인증서를 CloudFront에 연결할 수 있다.

---

## 12. CloudFront와 ACM 리전 주의사항

CloudFront에 사용할 ACM 인증서는 반드시 미국 동부 버지니아 북부 리전에 있어야 한다.

```text
US East (N. Virginia)
```

```text
us-east-1
```

이 점이 매우 중요하다.

예를 들어 S3 Bucket을 서울 리전에 만들었더라도, CloudFront에 연결할 ACM 인증서는 `us-east-1` 리전에서 발급해야 한다.

정리하면 다음과 같다.

```text
S3 Bucket Region: ap-northeast-2 가능
```

```text
CloudFront: Global Service
```

```text
ACM for CloudFront: us-east-1 필요
```

Day 06에서 실제 HTTPS를 적용할 때 이 부분을 반드시 확인해야 한다.

---

## 13. CloudFront Alternate Domain Name 개념

CloudFront 배포에는 기본 도메인이 자동으로 생성된다.

예를 들면 다음과 같다.

```text
dxxxxxxxxxxxxx.cloudfront.net
```

하지만 사용자는 보통 이런 주소로 접속하지 않는다.

대신 다음과 같은 사용자 지정 도메인을 사용한다.

```text
www.example.com
```

```text
example.com
```

CloudFront에서 사용자 지정 도메인을 사용하려면 Alternate Domain Name을 추가해야 한다.

Alternate Domain Name은 CloudFront 배포에 연결할 사용자 지정 도메인 이름이다.

예를 들어 CloudFront 배포에 다음 값을 추가할 수 있다.

```text
www.example.com
```

```text
example.com
```

그리고 Route 53에서 해당 도메인을 CloudFront 배포로 연결한다.

```text
www.example.com → CloudFront Distribution
```

```text
example.com → CloudFront Distribution
```

이때 CloudFront에는 해당 도메인을 포함하는 ACM 인증서도 연결되어 있어야 한다.

---

## 14. Route 53과 CloudFront 연결 흐름

사용자가 사용자 지정 도메인으로 접속할 때의 흐름은 다음과 같다.

```text
1. 사용자가 브라우저에 www.example.com 입력
2. Route 53이 www.example.com의 DNS 레코드를 확인
3. Route 53이 CloudFront 배포로 연결
4. CloudFront가 요청을 처리
5. CloudFront가 필요한 경우 S3 Bucket에서 파일 조회
6. 사용자에게 정적 웹사이트 콘텐츠 반환
```

전체 흐름은 다음과 같다.

```text
User
↓
Route 53
↓
CloudFront
↓
S3 Bucket
```

HTTPS까지 포함하면 다음과 같다.

```text
User
↓ HTTPS
Route 53
↓
CloudFront + ACM Certificate
↓
S3 Bucket
```

---

## 15. Day 05 정리

Day 05에서는 실제 도메인을 연결하기 전에 Route 53과 ACM의 핵심 개념을 정리했다.

핵심 내용은 다음과 같다.

- Route 53은 AWS의 DNS 서비스이다.
- Route 53을 사용한다고 해서 CloudFront가 필수인 것은 아니다.
- CloudFront는 Route 53이 연결할 수 있는 대상 중 하나이다.
- 이번 프로젝트에서는 정적 웹사이트를 안전하고 빠르게 제공하기 위해 CloudFront를 함께 사용한다.
- Hosted Zone은 특정 도메인의 DNS 레코드를 관리하는 공간이다.
- Hosted Zone 안에서 `www`, `api`, `mail`, `blog` 같은 하위 도메인 이름을 관리할 수 있다.
- 하위 도메인을 만든다는 것은 실제 서버를 만드는 것이 아니라, 해당 이름에 대한 DNS 레코드를 만드는 것이다.
- DNS Record는 도메인이 어떤 대상을 가리키는지 정의한다.
- Root Domain을 CloudFront에 연결할 때는 Route 53 Alias Record를 사용할 수 있다.
- ACM은 HTTPS 적용을 위한 SSL/TLS 인증서를 관리하는 서비스이다.
- CloudFront에서 사용할 ACM 인증서는 반드시 `us-east-1` 리전에서 발급해야 한다.
- CloudFront에서 사용자 지정 도메인을 사용하려면 Alternate Domain Name과 ACM 인증서가 필요하다.

---

## 16. Next Step

Day 06에서는 도메인이 준비되었다고 가정하고 실제 사용자 지정 도메인과 HTTPS 적용 흐름을 정리한다.

예상 작업은 다음과 같다.

- Route 53 Hosted Zone 생성 또는 확인
- ACM 인증서 요청
- DNS Validation 진행
- CloudFront에 Alternate Domain Name 추가
- CloudFront에 ACM 인증서 연결
- Route 53 Alias Record 생성
- 사용자 지정 도메인 접속 확인

예상 최종 구조는 다음과 같다.

```text
User → Route 53 → CloudFront + ACM → S3 Bucket
```
