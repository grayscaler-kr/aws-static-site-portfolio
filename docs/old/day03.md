# Day 3 - S3 + CloudFront 정적 웹사이트 배포

## 1. 목표

이번 실습의 목표는 S3에 정적 파일을 업로드하고, CloudFront를 통해 해당 정적 페이지에 접근하는 구성을 만드는 것이었다.

구성 목표는 다음과 같았다.

```text
사용자
→ CloudFront 도메인
→ S3 버킷의 index.html
```

최종적으로는 CloudFront에서 제공하는 도메인을 통해 S3에 업로드한 index.html 페이지에 접근하는 것을 확인했다.

## 2. S3 버킷 생성 및 파일 업로드

### S3 버킷 생성

정적 페이지 파일을 저장하기 위해 S3 버킷을 생성했다.

예시 버킷명:

```text
grayscaler-static-website-bucket
```

S3에는 index.html 파일을 업로드했다.

```text
s3://grayscaler-static-website-bucket/index.html
```

### 중요한 점

이번 구성에서는 S3의 Static website hosting endpoint를 사용하는 방식이 아니라, CloudFront에서 S3 버킷을 Origin으로 연결하는 방식을 사용했다.


즉, 접근 방식은 아래가 아니다.

```text
S3 Static Website Hosting endpoint
```

이번 접근 방식은 아래에 가깝다.

```text
CloudFront
→ S3 REST API endpoint
→ Private S3 Bucket
```

## 3. CloudFront Distribution 생성

CloudFront Distribution을 생성하고, Origin으로 S3 버킷을 연결했다.

### Origin domain

CloudFront Origin에는 S3 버킷의 REST API endpoint 형태를 사용했다.

```text
grayscaler-static-website-bucket.s3.ap-northeast-2.amazonaws.com
```

### Default root object

루트 경로 /로 접근했을 때 index.html이 응답되도록 설정했다.

```text
Default root object: index.html
```

이 설정을 통해 아래 두 접근을 확인할 수 있다.

```text
https://CloudFront도메인/
https://CloudFront도메인/index.html
```

## 4. S3 권한 설정

### Object Ownership

S3 버킷은 Object Ownership 설정에서 다음 상태였다.

```text
Bucket owner enforced
```

이 설정이 적용되면 ACL 기반 권한 제어는 비활성화된다.

따라서 S3 객체 권한을 ACL로 열어주는 방식이 아니라, Bucket Policy를 통해 접근을 제어해야 한다.

### ACL Edit 버튼 비활성화

Permissions 화면에서 ACL 관련 Edit 버튼이 비활성화되어 있었는데, 이는 비정상 상황이 아니라 Bucket owner enforced 설정 때문에 발생한 정상 동작이다.

## 5. CloudFront OAC 설정

CloudFront에서 S3 Origin 접근 방식으로 OAC를 사용했다.

```text
Origin access control settings, recommended
```

이 방식은 S3 버킷을 public으로 열지 않고, CloudFront를 통해서만 접근할 수 있도록 하는 방식이다.

### 목표

```text
S3 Bucket: Public Access 차단
CloudFront: S3 접근 허용
사용자: CloudFront 도메인으로 접근
```

## 6. S3 Bucket Policy 설정

CloudFront가 S3 객체를 읽을 수 있도록 S3 Bucket Policy에 CloudFront Distribution을 허용하는 정책을 추가했다.


예시 형태:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipalReadOnly",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::grayscaler-static-website-bucket/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
        }
      }
    }
  ]
}
```

### 핵심 포인트

S3 Bucket Policy에는 s3:GetObject 권한만 추가한다.

```text
s3:GetObject
```

KMS 관련 권한인 아래 Action들은 S3 Bucket Policy에 넣을 수 없다.

```text
kms:Decrypt
kms:DescribeKey
```

이 권한들은 S3 버킷 정책이 아니라 KMS Key Policy에 설정해야 한다.

## 7. AccessDenied 트러블슈팅

설정 후 CloudFront 도메인으로 접근했지만 계속 AccessDenied가 발생했다.

확인했던 항목은 다음과 같다.

```text
index.html이 S3에 정상 업로드되어 있는지 확인
CloudFront Origin domain 확인
CloudFront Origin이 S3 REST endpoint인지 확인
OAC가 연결되어 있는지 확인
Bucket Policy에 CloudFront Distribution ARN이 들어가 있는지 확인
Default root object가 index.html인지 확인
CloudFront Invalidation이 Completed 상태인지 확인
```

대부분의 설정은 정상처럼 보였다.

## 8. SSE-KMS에서 SSE-S3로 변경한 이유

### 원인: SSE-KMS와 AWS managed key aws/s3

최종적으로 문제 원인은 S3 객체 암호화 방식이었다.

S3 버킷의 기본 암호화가 다음으로 설정되어 있었다.

```text
Server-side encryption with AWS Key Management Service keys, SSE-KMS
```

그리고 사용 중인 KMS key는 Customer managed key가 아니라 AWS managed key였다.

```text
aws/s3
```

### 왜 문제가 되었는가?

SSE-KMS로 암호화된 객체에 접근하려면 S3 접근 권한뿐만 아니라 KMS 복호화 권한도 필요하다.

CloudFront가 SSE-KMS로 암호화된 S3 객체를 읽으려면 핵심적으로 아래 권한이 필요할 수 있다.

```text
s3:GetObject
kms:Decrypt
```

경우에 따라 KMS Key 확인을 위해 아래 권한을 함께 다룰 수 있다.

```text
kms:DescribeKey
```

하지만 aws/s3 AWS managed key는 사용자가 Key Policy를 직접 수정해서 CloudFront에 권한을 추가하기 어렵다.


그래서 CloudFront는 S3 Bucket Policy상으로는 허용되어 있어도, 암호화된 객체를 복호화하지 못해 AccessDenied가 발생한 것으로 판단했다.

### 시도했지만 실패한 방법

처음에는 S3 Bucket Policy에 KMS 권한을 추가하려고 했다.

예시:

```json
{
  "Action": [
    "kms:Decrypt",
    "kms:DescribeKey"
  ]
}
```

하지만 S3 Bucket Policy에 해당 권한을 넣으면 아래와 같은 오류가 발생했다.

```text
Unsupported Action In Policy:
The action kms:Decrypt is not supported for the resource-based policy attached to resource type S3 Bucket.

Unsupported Action In Policy:
The action kms:DescribeKey is not supported for the resource-based policy attached to resource type S3 Bucket.
```

### 이유

kms:Decrypt, kms:DescribeKey는 S3 Bucket Policy에 넣는 권한이 아니다.

이 권한들은 KMS Key에 대한 권한이므로 KMS Key Policy 또는 관련 IAM Policy에서 다뤄야 한다.

### 선택지: Customer managed KMS key 생성

정석적으로 SSE-KMS를 유지하려면 다음 방식으로 진행할 수 있다.

```text
KMS Customer managed key 생성
→ S3 기본 암호화 키를 해당 KMS key로 변경
→ KMS Key Policy에 CloudFront 접근 권한 추가
→ 객체 재업로드 또는 재암호화
→ CloudFront Invalidation
```

하지만 KMS Customer managed key 생성 과정은 별도의 단계가 많고, Key Policy까지 함께 다뤄야 한다.

단기간에 정적 페이지 배포 실습을 마무리하는 목적에서는 진도가 너무 늦어질 수 있다고 판단했다.

### 최종 해결: SSE-S3로 변경

이번 실습에서는 빠른 완성을 위해 S3 기본 암호화 방식을 SSE-KMS에서 SSE-S3로 변경했다.


변경 전:

```text
SSE-KMS
KMS key: aws/s3
```

변경 후:

```text
SSE-S3
Amazon S3 managed keys
```

조치한 작업

```text
S3 버킷 기본 암호화를 SSE-S3로 변경
index.html 파일 재업로드
버전 관리로 인해 새 버전 생성 확인
CloudFront Invalidation 새로 생성
CloudFront 도메인으로 접근 확인
```

SSE-S3는 S3가 관리하는 키로 서버 측 암호화를 수행하는 방식이다. 별도의 KMS Key Policy를 직접 관리하지 않아도 되므로 이번 실습 목적에는 더 단순했다.

### 중요한 점

S3 버킷의 기본 암호화 설정을 변경해도 기존 객체가 자동으로 새 암호화 방식으로 바뀌는 것은 아니다.

따라서 기존 index.html을 다시 업로드하여 객체 암호화 상태를 갱신했다.

## 9. CloudFront Invalidation

파일을 재업로드하고 암호화 방식을 변경한 뒤, CloudFront 캐시를 새로 고치기 위해 Invalidation을 생성했다.


예시 경로:

```text
/*
```

또는 특정 파일만 무효화할 수도 있다.

```text
/index.html
```

Invalidation 상태가 다음과 같이 바뀐 후 다시 접근했다.

```text
Completed
```

실습에서는 `/*`로 전체 무효화를 사용해도 괜찮지만, 운영 환경에서는 필요한 파일만 지정하는 것이 더 안전하고 효율적이다.

### 이전 Invalidation 기록

이전에 생성된 Invalidation은 삭제할 필요가 없다.

Completed 상태의 Invalidation은 CloudFront의 작업 이력으로 남는 것이며, 이후 동작에 영향을 주지 않는다.

## 10. 최종 결과

최종적으로 CloudFront 도메인을 통해 S3에 업로드한 정적 페이지 접근에 성공했다.

확인한 URL 형태:

```text
https://CloudFront도메인/index.html
https://CloudFront도메인/
```

최종 구성은 다음과 같다.

```text
사용자
→ CloudFront
→ OAC
→ Private S3 Bucket
→ SSE-S3로 암호화된 index.html
```

## 11. 배운 점

### 1. S3 ACL이 비활성화된 것은 문제 상황이 아닐 수 있다

Bucket owner enforced 설정에서는 ACL을 사용하지 않고 Bucket Policy로 권한을 제어한다.


### 2. CloudFront OAC를 쓰면 S3 버킷을 public으로 열 필요가 없다

S3 버킷은 private으로 유지하고, CloudFront에만 읽기 권한을 줄 수 있다.


### 3. CloudFront AccessDenied는 Bucket Policy만의 문제가 아닐 수 있다

처음에는 Bucket Policy 문제라고 생각하기 쉽지만, 실제로는 다음 요소들도 함께 봐야 한다.

```text
Origin domain
OAC 연결 여부
Default root object
S3 객체 위치
CloudFront 배포 상태
CloudFront 캐시
S3 객체 암호화 방식
KMS Key 권한
```

### 4. SSE-KMS 사용 시 KMS 권한이 별도로 필요하다

SSE-KMS로 암호화된 S3 객체를 읽으려면 S3 권한 외에도 KMS 권한이 필요하다.

```text
s3:GetObject
kms:Decrypt
```

### 5. aws/s3 AWS managed key는 실습 확장에 한계가 있다

AWS managed key인 aws/s3는 사용자가 Key Policy를 직접 수정하기 어렵다.

CloudFront OAC와 SSE-KMS를 함께 실습하려면 Customer managed key를 사용하는 편이 적절하다.

### 6. 빠른 배포 완성이 목표라면 SSE-S3가 더 단순하다

이번 실습에서는 정적 페이지 배포 자체가 목표였기 때문에 SSE-S3로 전환하여 문제를 해결했다.

```text
SSE-KMS: 심화 실습용
SSE-S3: 빠른 정적 배포 완성용
```

## 12. 최종 메모

이번 실습은 단순히 S3와 CloudFront를 연결하는 것에서 끝나지 않고, 실제로 배포 중 자주 마주칠 수 있는 AccessDenied 문제를 함께 경험했다.

특히 아래 흐름을 직접 확인한 것이 의미 있었다.

```text
S3 Bucket Policy는 정상
CloudFront OAC도 정상
하지만 SSE-KMS aws/s3 키 때문에 AccessDenied 발생
```

최종적으로는 SSE-S3로 전환하여 빠르게 문제를 해결했고, SSE-KMS는 추후 Customer managed key를 사용하는 심화 실습으로 남겨두었다.

## 13. 다음 단계

다음 실습에서는 CloudFront의 캐시 동작과 에러 페이지 설정을 다룬다.

예정 작업:

```text
error.html 업로드
CloudFront Custom Error Response 설정
index.html 수정 후 재업로드
CloudFront Invalidation 테스트
S3 직접 접근 차단 확인
```
