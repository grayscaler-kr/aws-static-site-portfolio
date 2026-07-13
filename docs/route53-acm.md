# Route 53, ACM 및 HTTPS 구성

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

## 15. 최종 메모

실제 도메인을 연결하기 전에 Route 53과 ACM의 핵심 개념을 정리했다.

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

## 16. 실제 구축

별도의 도메인 등록 기관(예: 가비아)에서 도메인을 등록한 후, Route 53과 ACM을 이용하여 사용자 지정 도메인과 HTTPS를 실제로 적용하는 과정을 진행한다.

### 16.1 Hosted Zone 생성 

AWS Management Console에서 **Route 53 → Hosted zones** 메뉴로 이동한다.
이후 Create hosted zone 을 선택하면 Hosted Zone 생성 화면으로 넘어간다.

Hosted Zone 생성 화면에서 다음 항목을 입력한다.
- Domain name : `grayscaler.dev`
- Type : Public hosted zone

Public hosted zone은 인터넷에서 접근 가능한 도메인의 DNS 정보를 관리하기 위한 Hosted Zone이다.

이번 프로젝트에서는 가비아에서 등록한 `grayscaler.dev`를 인터넷에 공개하기 위해 Public hosted zone을 사용하였다.

이후 Create hosted zone을 선택하면 Hosted Zone이 정상적으로 생성된다.

### 16.2 NS / SOA 자동 생성

Public Hosted Zone을 생성하면 Route 53은 기본적으로 NS(Name Server)와 SOA(Start of Authority) 레코드를 자동 생성한다.

- NS : 해당 도메인의 DNS 질의를 처리할 네임서버 정보
- SOA : DNS Zone의 관리 정보 및 동기화 정책을 관리하는 기본 레코드

이번 프로젝트에서는 Hosted Zone 생성과 동시에 NS 레코드 1개(4개의 네임서버 정보 포함)와 SOA 레코드가 자동 생성되는 것을 확인하였다.

이후 가비아의 네임서버를 Route 53의 NS 정보로 변경하여 DNS 관리 권한을 AWS로 이전할 예정이다.

### 16.3 가비아 네임서버(NS) 변경

가비아에 로그인 후 My가비아에서 **도메인 통합 관리툴**을 선택한다.

Gabia 도메인 통합 관리에서 **도메인 관리 → 전체 도메인 → grayscaler.dev** 선택한다.

도메인 상세에서 **네임서버/DNS호스트/DNSSEC** 선택한다.

네임서버에서 **설정** 선택한다.

기존 가비아에서 부여한 네임서버를 지우고 Route 53에서 생성한 Public hosted zone의 NS 레코드에 있는 aws 관리 네임서버를 입력한다.
등록된 네임서버는 아래와 같다.

- ns-1977.awsdns-55.co.uk
- ns-1020.awsdns-63.net
- ns-1169.awsdns-18.org
- ns-341.awsdns-42.com

가비아에서는 네임서버 변경할 때 휴대전화 또는 이메일 통한 소유자 인증이 필요하니 기호에 맞춰서 인증 진행한다.
소유자 인증 완료하면 **적용** 버튼이 활성화되니 선택한다.

변경 신이 완료되면 변경 결과를 확인한다.

#### 운영 고려 사항

Hosted Zone을 생성하는 것만으로는 인터넷에서 Route 53을 사용하지 않는다.

도메인 등록 기관의 네임서버를 Route 53의 NS 정보로 변경해야 해당 도메인의 DNS 질의가 Route 53으로 전달된다.

### 16.4 DNS 위임 확인

네임서버 변경 후 `nslookup`을 이용하여 DNS 위임이 정상적으로 이루어졌는지 확인하였다.

```bash
nslookup -type=ns grayscaler.dev
```

응답 결과 Route 53에서 생성된 네임서버 정보가 반환되는 것을 확인하였다.

이를 통해 `grayscaler.dev`의 DNS 관리 권한이 Route 53으로 정상 위임되었음을 확인하였다.

DNS 위임이 완료된 이후에는 Route 53에서 생성하는 DNS 레코드가 인터넷에 반영된다.

이후 ACM 인증서 발급 및 DNS Validation을 진행할 예정이다.

### 16.5 ACM 인증서 요청

CloudFront에 사용자 지정 도메인과 HTTPS를 적용하기 위한 준비 단계로 ACM에서 Public Certificate를 요청한다.

CloudFront에서 사용할 ACM 인증서는 반드시 `us-east-1`(US East, N. Virginia) 리전에 생성되어 있어야 하므로, AWS Management Console의 리전을 `us-east-1`로 변경한 후 작업을 진행하였다.

-   AWS Certificate Manager (ACM)로 이동한 후 **Request a certificate**를 선택한다.
-   Certificate type은 기본값인 **Request a public certificate**를 선택한다.
-   Domain names에 다음 두 도메인을 입력한다.
    -   `grayscaler.dev`
    -   `*.grayscaler.dev`
-   Allow export는 **Disable export**를 선택한다.
-   Validation method는 **DNS validation**을 선택한다.
-   Key algorithm은 **RSA 2048**을 선택한다.
-   설정을 확인한 후 **Request**를 선택하여 인증서를 요청한다.

인증서 요청 직후에는 도메인 소유권 검증이 완료되지 않았으므로 인증서 상태가 `Pending validation`으로 표시된다.

#### 운영 고려 사항

CloudFront는 글로벌 서비스이지만, CloudFront의 사용자 HTTPS 인증서로 사용할 ACM 인증서는 반드시 `us-east-1` 리전에 존재해야 한다.

다른 리전에 생성한 ACM 인증서는 CloudFront의 사용자 지정 인증서로 선택할 수 없다.

**Disable export**를 선택하면 인증서의 개인 키를 AWS 외부로 내보낼 수 없다. 이번 프로젝트에서는 인증서를 CloudFront에서만 사용할 예정이므로 개인 키를 외부 서버에 배포할 필요가 없으며, AWS가 인증서와 개인 키를 관리하도록 구성하였다.

**DNS validation**은 ACM에서 제공하는 CNAME 레코드를 DNS에 등록하여 도메인의 제어 권한을 증명하는 방식이다. Route 53에서 DNS를 관리하는 경우 ACM 콘솔을 통해 검증용 CNAME 레코드를 자동으로 생성할 수 있다.

**RSA 2048**은 인증서에 사용되는 공개키·개인키 쌍의 암호 알고리즘이다. 다양한 브라우저와 클라이언트에서 폭넓게 지원되므로 이번 프로젝트에서는 호환성을 고려해 RSA 2048을 선택하였다.

### 16.6 DNS Validation

ACM 인증서 요청 후 인증서 상태가 `Pending validation`으로 표시되는 것을 확인하였다.

인증서 요청 대상은 다음과 같다.

-   `grayscaler.dev`
-   `*.grayscaler.dev`
    
ACM은 도메인 소유권 검증을 위해 CNAME 레코드의 이름과 값을 생성한다.

-   CNAME name: Route 53에 생성할 레코드 이름
-   CNAME value: 해당 CNAME 레코드가 연결할 ACM 검증 주소

이번 프로젝트에서는 `grayscaler.dev`의 DNS를 Route 53에서 관리하고 있으므로, ACM의 **Create records in Route 53** 기능을 이용해 검증용 CNAME 레코드를 자동으로 생성하였다.

Route 53 Hosted Zone에서 NS, SOA 레코드와 함께 ACM 검증용 CNAME 레코드가 추가된 것을 확인하였다.

CNAME 레코드 생성 직후에는 인증서 상태가 `Pending validation`으로 유지되며, ACM이 DNS 레코드를 확인하면 상태가 `Issued`로 변경된다.

#### 운영 고려 사항

DNS Validation에 사용되는 CNAME 레코드는 인증서가 발급된 이후에도 삭제하지 않는다.

검증용 CNAME 레코드가 유지되어야 ACM에서 인증서를 자동 갱신할 수 있다.

### 16.7 인증서 발급 확인

ACM에서 `grayscaler.dev`와 `*.grayscaler.dev`를 대상으로 Public Certificate를 요청하였다.

DNS Validation을 위해 ACM이 제공한 CNAME name/value 쌍을 Route 53 Hosted Zone에 등록하였으며, ACM이 해당 DNS 레코드를 확인한 뒤 인증서 상태가 `Pending validation`에서 `Issued`로 변경되었다.

도메인별 검증 상태도 모두 `Success`로 표시되는 것을 확인하였다.

이를 통해 `grayscaler.dev`에 대한 도메인 제어 권한 검증이 완료되었고, CloudFront에서 사용할 수 있는 ACM 인증서가 정상적으로 발급되었다.

#### 운영 고려 사항

현재 인증서는 아직 CloudFront에 연결하지 않았으므로 `In use` 상태는 `No`로 표시된다.

### 16.8 CloudFront 사용자 지정 도메인 및 ACM 인증서 연결

기존 CloudFront 배포에 사용자 지정 도메인과 ACM 인증서를 연결하였다.

CloudFront 배포의 General > Edit 선택 후 편집 화면에서 다음 항목으로 구성하였다.

- Alternate domain name: `grayscaler.dev`
- Custom SSL certificate: `grayscaler.dev`와 `*.grayscaler.dev`를 포함한 ACM 인증서
- Security policy: `TLSv1.2_2021`
- Supported HTTP versions: HTTP/2, HTTP/3

CloudFront는 Alternate Domain Name에 추가한 도메인이 연결된 인증서의 도메인 범위에 포함되는지 확인한다.

이번 프로젝트에서 사용한 ACM 인증서에는 발급 당시 `grayscaler.dev`가 직접 포함되어 있으므로 CloudFront에 정상적으로 연결할 수 있었다.

설정 저장 후 CloudFront의 **Last modified** 항목이 `Deploying`으로 표시되었으며, 이는 변경 사항이 CloudFront 엣지 로케이션에 배포되고 있음을 의미한다.

일정 시간이 지난 후 `Deploying` 표시가 사라지고 마지막 수정 시간이 표시되었으며, 이를 통해 설정 배포가 완료되었음을 확인하였다.

#### 운영 고려 사항

CloudFront에 Alternate Domain Name과 ACM 인증서를 연결하는 것만으로는 DNS 연결이 완료되지 않는다.

배포가 완료된 후 Route 53에서 `grayscaler.dev`를 CloudFront 배포로 연결하는 Alias 레코드를 별도로 생성해야 한다.

### 16.9 HTTPS 리디렉션 설정 확인

CloudFront 배포의 **Behaviors**에서 Default behavior를 확인하였다.

Viewer protocol policy가 **Redirect HTTP to HTTPS**로 설정되어 있어 별도의 변경은 진행하지 않았다.

이 설정에 따라 사용자가 HTTP로 접속하더라도 CloudFront가 HTTPS 주소로 리디렉션한다.

또한 정적 웹사이트 조회만 필요하므로 Allowed HTTP methods는 `GET, HEAD`로 유지하였다.

#### 운영 고려 사항

CloudFront에 사용자 지정 도메인과 ACM 인증서를 연결하고 HTTPS 리디렉션을 설정하더라도, DNS에서 해당 도메인을 CloudFront 배포로 연결하지 않으면 사용자 지정 도메인으로 접속할 수 없다.

다음 단계에서는 Route 53에 A Alias 레코드를 생성하여 `grayscaler.dev`를 CloudFront 배포로 연결한다.

### 16.10 Route 53 Alias 레코드 생성

`grayscaler.dev`를 기존 CloudFront 배포로 연결하기 위해 Route 53 Hosted Zone에 Alias 레코드를 생성하였다.

생성한 레코드는 다음과 같다.

- A Alias: IPv4 요청을 CloudFront 배포로 연결
- AAAA Alias: IPv6 요청을 CloudFront 배포로 연결

두 레코드 모두 다음 CloudFront 배포를 대상으로 설정하였다.

- `daxb744d4ejyw.cloudfront.net`

Record name은 비워 두어 Hosted Zone의 루트 도메인인 `grayscaler.dev`에 레코드가 생성되도록 하였다.

Routing policy는 **Simple routing**을 사용하였으며, Evaluate target health는 비활성화하였다.

#### 운영 고려 사항

CloudFront 배포에는 Route 53 레코드 이름과 일치하는 Alternate Domain Name이 미리 등록되어 있어야 한다.

이번 프로젝트에서는 CloudFront의 Alternate Domain Name에 `grayscaler.dev`를 등록한 후, Route 53에서 A 및 AAAA Alias 레코드를 생성하였다.

### 16.11 최종 접속 및 HTTPS 검증

Route 53 Alias 레코드 생성 후 DNS 조회와 브라우저 접속을 통해 전체 구성을 검증하였다.

다음 명령으로 `grayscaler.dev`의 DNS 응답을 확인하였다.

```bash
nslookup grayscaler.dev
```

조회 결과 CloudFront에서 사용하는 IPv4 및 IPv6 주소가 반환되는 것을 확인하였다.

브라우저에서는 다음 항목을 확인하였다.

-   `https://grayscaler.dev` 정상 접속
-   `http://grayscaler.dev` 접속 시 HTTPS로 자동 전환
-   S3에 저장된 정적 웹페이지 정상 출력
-   인증서 발급 대상이 `grayscaler.dev`로 표시
-   인증서 발급 기관이 Amazon 계열 인증 기관으로 표시
    

이를 통해 다음 연결이 정상적으로 동작함을 확인하였다.

```text
User
  ↓ DNS Query
Route 53
  ↓ A / AAAA Alias
CloudFront + ACM
  ↓ OAC
S3 Bucket (Private)
```

#### 검증 시 참고 사항

초기 인증서 확인 시 로컬 광고 차단 프로그램인 Unicorn Pro가 HTTPS 트래픽에 개입하여 `Unicorn Root CA`가 표시되었다.

Unicorn Pro를 일시 중지하고 브라우저 데이터를 삭제한 후 다시 확인한 결과, 인증서 발급 기관이 `Amazon RSA 2048 M01`로 정상 표시되었다.

이를 통해 AWS 설정 문제가 아니라 로컬 HTTPS 필터링 프로그램의 영향이었음을 확인하였으며, CloudFront에 연결한 ACM 인증서가 실제 HTTPS 요청에 정상 적용되었음을 검증하였다.

#### 트러블슈팅: www 도메인 접속 실패

Route 53에 `www.grayscaler.dev`의 A/AAAA Alias 레코드를 생성했지만 페이지가 정상적으로 열리지 않았다.

원인은 CloudFront 배포의 Alternate Domain Name에 `www.grayscaler.dev`가 등록되어 있지 않았기 때문이다.

Route 53 레코드는 DNS 요청을 CloudFront 배포로 전달하는 역할을 하지만, CloudFront가 해당 호스트명의 요청을 처리하려면 Alternate Domain Name에도 동일한 도메인이 등록되어 있어야 한다.

CloudFront Alternate Domain Name에 `www.grayscaler.dev`를 추가한 후 정상 접속되는 것을 확인하였다.
