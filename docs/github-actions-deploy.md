# GitHub Actions 기반 자동 배포

## 1. 자동 배포 목적

기존에는 정적 웹사이트 파일을 수정한 후 다음 작업을 수동으로 수행하였다.

1. 수정한 파일을 S3 Bucket에 업로드
2. CloudFront Cache Invalidation 생성
3. 웹사이트에서 최신 콘텐츠 반영 여부 확인

이 방식은 작업 단계가 반복되며, 파일 업로드 또는 캐시 무효화를 누락할 가능성이 있다.

이를 개선하기 위해 GitHub의 `main` 브랜치에 `website/` 디렉토리 변경 사항을 Push하면 다음 작업이 자동으로 실행되도록 구성하였다.

- GitHub Actions Workflow 실행
- AWS IAM Role을 통한 임시 자격 증명 발급
- S3 Bucket에 정적 웹사이트 파일 동기화
- CloudFront Cache Invalidation 생성
- 최신 콘텐츠 반영

자동 배포를 통해 반복 작업을 줄이고 배포 과정의 일관성과 재현성을 확보하는 것을 목표로 하였다.

---

## 2. 전체 구조

자동 배포의 전체 흐름은 다음과 같다.

```text
Developer
   │
   └── Git Push
          │
          ▼
GitHub Repository
   │
   └── main 브랜치의 website/ 변경 감지
          │
          ▼
GitHub Actions
   │
   ├── GitHub OIDC Token 발급
   │
   ▼
AWS STS
   │
   ├── IAM Trust Policy 검증
   ├── IAM Role Assume
   └── 임시 AWS 자격 증명 발급
          │
          ▼
GitHubActionsStaticSiteDeployRole
   │
   ├── S3 Bucket 파일 동기화
   └── CloudFront Cache Invalidation 생성
          │
          ▼
CloudFront
   │
   └── 최신 S3 콘텐츠 제공
          │
          ▼
grayscaler.dev
www.grayscaler.dev
```

GitHub Actions가 AWS에 접근할 때 장기 Access Key를 사용하지 않고, GitHub OIDC Token과 AWS STS를 통해 임시 자격 증명을 발급받도록 구성하였다.

---

## 3. GitHub OIDC Provider 구성

GitHub Actions에서 AWS IAM Role을 사용할 수 있도록 IAM에 GitHub OIDC Provider를 등록하였다.

### 설정값

-   Provider type: OpenID Connect
-   Provider URL: `https://token.actions.githubusercontent.com`
-   Audience: `sts.amazonaws.com`

GitHub Actions Workflow는 실행 시 GitHub로부터 OIDC Token을 발급받는다.

AWS는 등록된 OIDC Provider를 통해 토큰의 발급자가 GitHub인지 확인하고, Audience 값이 `sts.amazonaws.com`인지 검증한다.

`sts.amazonaws.com`은 해당 토큰이 AWS Security Token Service를 대상으로 발급되었다는 것을 의미한다.

AWS STS는 GitHub OIDC Token과 IAM Role의 Trust Policy를 검증한 후, 조건이 일치하면 Workflow에 임시 AWS 자격 증명을 발급한다.

임시 자격 증명은 다음 값으로 구성된다.

-   Access Key ID
-   Secret Access Key
-   Session Token

이 값은 Workflow 실행 중에만 사용되며, 장기 Access Key를 GitHub Secrets에 저장하지 않아도 된다.

---

## 4. IAM Role Trust Policy

GitHub Actions가 사용할 전용 IAM Role을 생성하였다.

### Role 정보

-   Role name: `GitHubActionsStaticSiteDeployRole`
-   Trusted entity type: Web identity
-   Identity provider: `token.actions.githubusercontent.com`
-   Audience: `sts.amazonaws.com`
-   GitHub organization: `grayscaler-kr`
-   GitHub repository: `aws-static-site-portfolio`
-   GitHub branch: `main`

생성된 Trust Policy는 GitHub OIDC Provider가 해당 Role을 사용할 수 있도록 허용한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:grayscaler-kr/aws-static-site-portfolio:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

### Trust Policy의 역할

Trust Policy는 해당 IAM Role을 누가 맡을 수 있는지를 정의한다.

이번 구성에서는 다음 조건을 모두 만족하는 경우에만 Role을 Assume할 수 있다.

-   GitHub가 발급한 OIDC Token일 것
-   Audience가 `sts.amazonaws.com`일 것
-   GitHub 저장소가 `grayscaler-kr/aws-static-site-portfolio`일 것
-   실행 브랜치가 `main`일 것

이를 통해 다른 GitHub 저장소나 다른 브랜치에서 해당 IAM Role을 사용하는 것을 제한하였다.

---

## 5. 최소 권한 정책

GitHub Actions가 배포 작업에 필요한 권한만 사용할 수 있도록 Inline Policy를 생성하였다.

### Policy 정보

-   Policy name: `StaticSiteDeployPolicy`
-   Policy type: Inline policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListTargetBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::grayscaler-static-website-bucket"
    },
    {
      "Sid": "DeployWebsiteObjects",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::grayscaler-static-website-bucket/*"
    },
    {
      "Sid": "InvalidateCloudFrontCache",
      "Effect": "Allow",
      "Action": [
        "cloudfront:CreateInvalidation"
      ],
      "Resource": "arn:aws:cloudfront::<AWS_ACCOUNT_ID>:distribution/E3DJWU1GIGGRVN"
    }
  ]
}
```

### 권한별 용도

#### `s3:ListBucket`

S3 Bucket에 저장된 객체 목록을 조회하기 위해 사용한다.

`aws s3 sync` 명령은 로컬 파일과 S3 객체를 비교해야 하므로 Bucket 목록 조회 권한이 필요하다.

#### `s3:GetBucketLocation`

배포 대상 S3 Bucket의 리전을 확인하기 위해 사용한다.

#### `s3:PutObject`

`website/` 디렉토리의 파일을 S3 Bucket에 업로드하거나 기존 객체를 갱신하기 위해 사용한다.

#### `s3:DeleteObject`

`aws s3 sync` 명령의 `--delete` 옵션을 사용할 때, 로컬 `website/` 디렉토리에서 삭제된 파일을 S3에서도 삭제하기 위해 사용한다.

#### `cloudfront:CreateInvalidation`

S3에 새로운 파일이 배포된 후 CloudFront에 남아 있는 기존 캐시를 무효화하기 위해 사용한다.

### 최소 권한 적용

`AdministratorAccess` 또는 `AmazonS3FullAccess`와 같은 광범위한 AWS 관리형 정책은 사용하지 않았다.

대신 다음 리소스로 권한 범위를 제한하였다.

-   특정 S3 Bucket
-   특정 S3 Bucket 내부 객체
-   특정 CloudFront Distribution

이를 통해 GitHub Actions가 자동 배포에 필요한 작업만 수행할 수 있도록 최소 권한 원칙을 적용하였다.

---

## 6. GitHub Actions Workflow

자동 배포 Workflow는 다음 경로에 생성하였다.

```text
.github/
└── workflows/
    └── deploy.yml
```

Workflow는 `main` 브랜치에 다음 파일이 변경되어 Push될 때 실행된다.

-   `website/**`
-   `.github/workflows/deploy.yml`

```yaml
name: Deploy static website

on:
  push:
    branches:
      - main
    paths:
      - "website/**"
      - ".github/workflows/deploy.yml"

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v5

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::<AWS_ACCOUNT_ID>:role/GitHubActionsStaticSiteDeployRole
          aws-region: ap-northeast-2

      - name: Verify AWS identity
        run: aws sts get-caller-identity

      - name: Deploy website to S3
        run: |
          aws s3 sync website/ s3://grayscaler-static-website-bucket/ \
            --delete

      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id E3DJWU1GIGGRVN \
            --paths "/*"
```

---

## 7. Workflow 주요 설정

### 실행 조건

```yaml
on:
  push:
    branches:
      - main
    paths:
      - "website/**"
      - ".github/workflows/deploy.yml"
```

모든 Push에서 Workflow가 실행되지 않도록 대상 브랜치와 변경 경로를 제한하였다.

README 또는 문서 파일만 변경된 경우에는 자동 배포가 실행되지 않는다.

### GitHub Token 권한

```yaml
permissions:
  id-token: write
  contents: read
```

#### `id-token: write`

GitHub Actions가 AWS에 전달할 OIDC Token을 발급받기 위해 필요하다.

AWS 리소스에 쓰기 권한을 부여하는 설정이 아니라, GitHub OIDC Token을 요청할 수 있도록 허용하는 설정이다.

#### `contents: read`

Workflow가 저장소 내용을 Checkout할 수 있도록 읽기 권한을 부여한다.

### Repository Checkout

```yaml
- name: Checkout repository
  uses: actions/checkout@v5
```

GitHub-hosted Runner에 저장소 내용을 내려받는다.

이후 단계에서 `website/` 디렉토리에 접근하려면 Checkout 과정이 먼저 필요하다.

### AWS 자격 증명 설정

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v6
```

GitHub OIDC Token을 이용하여 `GitHubActionsStaticSiteDeployRole`을 Assume한다.

Role Assume에 성공하면 Workflow 실행 중 사용할 임시 AWS 자격 증명이 설정된다.

AWS Access Key와 Secret Access Key를 GitHub Secrets에 별도로 저장하지 않았다.

### AWS 신원 검증

```yaml
- name: Verify AWS identity
  run: aws sts get-caller-identity
```

현재 Workflow가 어떤 AWS 계정과 IAM Role로 인증되었는지 확인한다.

실행 결과에서 다음과 같은 ARN을 확인하였다.

```text
arn:aws:sts::<AWS_ACCOUNT_ID>:assumed-role/GitHubActionsStaticSiteDeployRole/GitHubActions
```

이를 통해 GitHub Actions가 OIDC를 이용해 IAM Role을 정상적으로 Assume한 것을 검증하였다.

---

## 8. S3 Sync

S3 배포에는 다음 명령을 사용하였다.

```bash
aws s3 sync website/ s3://grayscaler-static-website-bucket/ --delete
```

이 명령은 로컬 저장소의 `website/` 디렉토리와 S3 Bucket을 비교하여 변경된 파일만 업로드한다.


### 동작 방식

일반적인 로컬 환경에서 `aws s3 sync`는 파일 크기와 수정 시간을 비교하여 다음과 같이 동작한다.

- 새 파일: S3에 업로드
- 수정된 파일: S3 객체 갱신
- 변경되지 않은 파일: 업로드 생략
- 로컬에서 삭제된 파일: `--delete` 옵션에 의해 S3에서도 삭제

다만 GitHub Actions에서는 Workflow 실행마다 새로운 Runner에서 저장소를 Checkout하므로 파일 수정 시간이 새로 설정될 수 있다. 이에 따라 실제 내용이 변경되지 않은 파일도 다시 업로드될 수 있음을 확인하였다.

### 운영 고려 사항

`--delete` 옵션을 사용하면 GitHub 저장소의 `website/` 디렉토리와 S3 Bucket의 객체 상태를 동일하게 유지할 수 있다.

다만 로컬 저장소에서 실수로 파일을 삭제한 상태로 Push하면 S3에서도 해당 객체가 삭제될 수 있으므로 주의가 필요하다.

이번 프로젝트에서는 S3 Versioning이 활성화되어 있어 삭제 또는 덮어쓰기 발생 시 이전 객체 버전을 이용한 복구가 가능하다.

---

## 9. CloudFront Cache Invalidation

S3 객체가 갱신되어도 CloudFront Edge Cache에 이전 콘텐츠가 남아 있으면 사용자는 수정 전 페이지를 볼 수 있다.

이를 방지하기 위해 S3 배포가 완료된 후 다음 명령을 실행하였다.

```bash
aws cloudfront create-invalidation \
  --distribution-id E3DJWU1GIGGRVN \
  --paths "/*"
```

### 설정 내용

-   Distribution ID: `E3DJWU1GIGGRVN`
-   Invalidation path: `/*`

`/*`를 사용하여 CloudFront Distribution에 캐시된 모든 경로를 무효화하였다.

GitHub Actions 실행 로그에서 Invalidation 생성 결과와 Invalidation ID가 반환되는 것을 확인하였다.

CloudFront 콘솔에서는 생성된 Invalidation의 상태가 다음 순서로 변경되는 것을 확인하였다.

```text
InProgress
→ Completed
```

---

## 10. 실제 배포 검증

자동 배포 동작을 검증하기 위해 `website/index.html`의 내용을 수정한 후 Commit 및 Push하였다.

### 검증 순서

1.  `website/index.html` 수정
2.  Git Commit
3.  `main` 브랜치로 Push
4.  GitHub Actions Workflow 자동 실행
5.  GitHub OIDC 기반 IAM Role Assume 확인
6.  S3 Sync 성공 확인
7.  CloudFront Invalidation 생성 확인
8.  S3 객체 최종 수정 시간 확인
9.  CloudFront Invalidation 완료 확인
10.  웹 브라우저에서 최신 콘텐츠 출력 확인

### 확인 결과

-   GitHub Actions Job 성공
-  `aws sts get-caller-identity` 성공
-  `website/` 디렉토리의 정적 콘텐츠가 S3 Bucket에 정상 동기화됨
-   S3 객체 최종 수정 시간 갱신 확인
-   CloudFront Invalidation 생성 및 완료 확인
-   `https://grayscaler.dev`에서 수정 내용 반영 확인
-   `https://www.grayscaler.dev`에서 수정 내용 반영 확인
-   HTTP 접속 시 HTTPS 리디렉션 후 최신 내용 출력 확인

이를 통해 다음 자동 배포 과정이 정상적으로 동작함을 확인하였다.

```text
Git Push
→ GitHub Actions
→ OIDC 인증
→ IAM Role Assume
→ S3 Sync
→ CloudFront Invalidation
→ 최신 콘텐츠 제공
```

---

## 11. 운영 고려 사항

### 장기 Access Key 미사용

GitHub Secrets에 장기 AWS Access Key와 Secret Access Key를 저장하지 않았다.

GitHub OIDC와 AWS STS를 이용해 Workflow 실행 시에만 임시 자격 증명을 발급받도록 구성하였다.

이를 통해 장기 자격 증명 유출과 수동 키 교체에 대한 운영 부담을 줄였다.

### Trust Policy 범위 제한

IAM Role의 Trust Policy를 특정 GitHub 저장소와 `main` 브랜치로 제한하였다.

다른 저장소 또는 다른 브랜치에서 발급된 OIDC Token은 해당 Role을 Assume할 수 없다.

### 최소 권한 적용

배포 Role에는 다음 작업만 허용하였다.

-   특정 S3 Bucket 조회
-   특정 S3 Bucket의 객체 업로드 및 삭제
-   특정 CloudFront Distribution의 Invalidation 생성

전체 S3 또는 CloudFront 리소스에 대한 광범위한 권한은 부여하지 않았다.

### 배포 순서

CloudFront Invalidation은 S3 Sync가 성공한 후에만 실행된다.

S3 배포 단계가 실패하면 이후 단계가 실행되지 않으므로, 캐시 무효화만 수행되는 상황을 방지할 수 있다.

### 전체 캐시 무효화

현재는 단순한 정적 웹사이트이고 파일 수가 적기 때문에 `/*` 경로를 사용하였다.

운영 규모가 커질 경우 모든 경로를 무효화하는 대신 변경된 파일 경로만 대상으로 지정하는 방법을 고려할 수 있다.

예:

```text
/index.html
/css/style.css
/js/app.js
```

### `--delete` 사용 주의

`--delete` 옵션은 저장소와 S3의 객체 상태를 일치시키는 데 유용하지만, 잘못된 파일 삭제가 S3에도 반영될 수 있다.

운영 환경에서는 다음 방법을 함께 고려할 수 있다.

-   S3 Versioning 유지
-   Pull Request 검토 후 `main` 병합
-   배포 전 테스트 단계 추가
-   삭제 대상 확인을 위한 Dry Run 수행
-   GitHub Environment 승인 절차 적용

### Workflow 실행 범위

현재 Workflow는 `main` 브랜치의 `website/` 또는 Workflow 파일이 변경된 경우에만 실행된다.

문서 파일 수정만으로 불필요한 S3 배포와 CloudFront Invalidation이 실행되지 않도록 경로를 제한하였다.

### GitHub Actions 환경에서의 S3 Sync 동작

GitHub Actions는 Workflow 실행마다 새로운 Runner에서 저장소를 Checkout한다.

이 과정에서 작업 디렉토리의 파일 수정 시간이 새로 설정되므로, 파일 내용이 변경되지 않았더라도 로컬 파일의 수정 시간이 기존 S3 객체보다 최신으로 판단될 수 있다.

`aws s3 sync`는 기본적으로 파일 크기와 수정 시간을 기준으로 업로드 여부를 판단하므로, 현재 구성에서는 `website/`의 파일이 매 실행마다 다시 업로드되는 것을 확인하였다.

`--size-only` 옵션을 사용할 경우 불필요한 업로드를 줄일 수 있지만, 내용이 변경되었음에도 파일 크기가 동일한 경우 변경 사항이 누락될 수 있다.

현재 프로젝트는 정적 파일 수와 크기가 작기 때문에 배포 안정성을 우선하여 기본 동작을 유지하였다.

---

## 12. 결과

GitHub Actions와 AWS IAM OIDC 연동을 통해 장기 Access Key 없이 자동 배포 환경을 구성하였다.

`main` 브랜치에 정적 웹사이트 변경 사항을 Push하면 다음 과정이 자동으로 실행된다.

-   IAM Role 임시 자격 증명 발급
-   S3 정적 콘텐츠 동기화
-   CloudFront Cache Invalidation 생성
-   최신 웹사이트 콘텐츠 반영

이를 통해 수동 배포 과정에서 발생할 수 있는 작업 누락을 줄이고, 반복 가능하고 일관된 배포 구조를 구성하였다.


