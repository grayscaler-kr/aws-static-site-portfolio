# Day 04 - CloudFront 캐시와 에러 페이지 설정

## 1. 목표

CloudFront를 통해 S3 정적 웹 사이트를 배포하고, 캐시 동작과 에러 페이지 처리 방식을 확인한다.

이번 실습에서는 다음 내용을 확인한다.

- CloudFront Custom Error Response 설정
- 잘못된 경로 접근 시 `error.html` 반환
- S3 원본 직접 접근 차단 확인
- `index.html` 수정 후 CloudFront 캐시 동작 확인
- Default root object 동작 확인
- CloudFront Invalidation을 통한 캐시 무효화
- S3 Versioning을 통한 객체 버전 확인

---

## 2. 현재 구성 확인

현재 정적 사이트는 다음 구조로 구성되어 있다.

- 정적 파일 저장소: Amazon S3
- CDN 및 배포 경로: Amazon CloudFront
- Origin 접근 방식: CloudFront를 통해 S3에 접근
- S3 직접 접근: 차단
- 기본 페이지: `index.html`
- 에러 페이지: `error.html`

CloudFront의 Default root object는 다음과 같이 설정되어 있다.

```text
index.html
```

따라서 CloudFront 도메인 루트 경로로 접근하면 `index.html`이 반환된다.

```text
https://CloudFront도메인/
```

## 3. error.html 업로드

잘못된 경로로 접근했을 때 보여줄 에러 페이지를 생성한다.

예시 파일:

```html
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <title>Error Page</title>
</head>
<body>
  <h1>404 Not Found</h1>
  <p>요청한 페이지를 찾을 수 없습니다.</p>
</body>
</html>
```

작성한 `error.html` 파일을 S3 버킷에 업로드한다.

업로드 후 S3 객체 목록에서 다음 파일이 존재하는지 확인한다.

```text
error.html
```

## 4. CloudFront Custom Error Response 설정

CloudFront에서 잘못된 경로 접근 시 사용자에게 `error.html`을 보여주도록 Custom Error Response를 설정한다.

설정값은 다음과 같다.

```text
HTTP error code: 403 Forbidden
Response page path: /error.html
HTTP Response code: 404 Not Found
Error caching minimum TTL: 10
```

### 설정 이유

S3 버킷을 직접 공개하지 않고 CloudFront를 통해서만 접근하도록 구성한 경우, 존재하지 않는 객체에 접근했을 때 S3가 `404 Not Found` 대신 `403 Forbidden`을 반환할 수 있다.

따라서 CloudFront에서는 `403 Forbidden` 에러를 잡아서 `/error.html`을 보여주도록 설정한다.

다만 사용자 입장에서는 존재하지 않는 페이지에 접근한 것이므로 최종 HTTP 응답 코드는 `404 Not Found`로 반환하도록 설정한다.

정리하면 다음과 같다.

```text
S3/CloudFront 원본 에러: 403 Forbidden
사용자에게 보여줄 페이지: /error.html
사용자에게 반환할 HTTP 상태 코드: 404 Not Found
```

## 5. 잘못된 경로 접근 테스트

존재하지 않는 경로로 접근하여 Custom Error Response가 정상 동작하는지 확인한다.

예시:

```text
https://CloudFront도메인/not-found.html
```

확인 결과:

```text
error.html 페이지가 표시됨
브라우저 응답 코드는 404 Not Found로 처리됨
```

이를 통해 CloudFront가 S3의 403 응답을 받아 사용자 정의 에러 페이지로 변환하는 것을 확인했다.

## 6. index.html 수정 및 재업로드

CloudFront 캐시 동작을 확인하기 위해 `index.html` 파일 내용을 수정한다.

예시 수정 내용:

```html
<h1>Day 04 CloudFront Cache Test</h1>
<p>index.html 파일을 수정한 뒤 CloudFront 캐시 동작을 확인합니다.</p>
```

수정한 `index.html` 파일을 S3에 다시 업로드한다.

S3 Versioning이 활성화되어 있으므로 같은 파일명으로 업로드해도 기존 객체가 삭제되는 것이 아니라 새로운 버전이 생성된다.

확인 항목:

```text
index.html 최신 버전 생성 여부
이전 버전 보존 여부
```

## 7. CloudFront 캐시 확인

S3에 수정된 `index.html`을 업로드한 뒤 CloudFront 도메인으로 접속하여 페이지 변경 사항이 바로 반영되는지 확인한다.

접속 URL:

```text
https://CloudFront도메인/
```

확인 결과:

```text
CloudFront 캐시 상태에 따라 이전 index.html이 보일 수 있음
또는 캐시가 만료된 경우 최신 index.html이 보일 수 있음
```

CloudFront는 S3에서 파일을 매번 새로 가져오는 것이 아니라, Edge Location에 캐시된 객체를 우선 반환한다.

따라서 S3에 파일을 새로 업로드해도 CloudFront에서는 일정 시간 동안 이전 파일이 보일 수 있다.

## 8. Default root object 캐시 확인

CloudFront의 Default root object가 `index.html`로 설정되어 있는 상태에서 아래 두 URL을 테스트한다.

```text
https://CloudFront도메인/
https://CloudFront도메인/index.html
```

확인 결과:

```text
/ 경로에서는 갱신된 index.html이 표시됨
/index.html 경로에서는 처음에 갱신 이전 index.html이 표시됨
이후 /index.html 새로고침 후 최신 index.html로 갱신됨
```

### 정리

CloudFront에서 `/`와 `/index.html`은 결과적으로 같은 `index.html`을 보여줄 수 있지만, 캐시 관점에서는 서로 다른 요청 경로로 취급될 수 있다.

```text
/              → Default root object에 의해 index.html 반환
/index.html    → index.html 객체 직접 요청
```

따라서 아래와 같은 상황이 발생할 수 있다.

```text
/              → 최신 index.html 표시
/index.html    → 이전 index.html 표시
```

이번 테스트에서는 `/index.html`을 새로고침한 뒤 최신 내용으로 갱신되었다.

이는 CloudFront 캐시 TTL이 만료되었거나, 브라우저 캐시가 재검증되면서 최신 응답을 받은 것으로 볼 수 있다.

## 9. Invalidation 생성

CloudFront 캐시에 남아 있는 이전 객체를 강제로 제거하기 위해 Invalidation을 생성한다.

CloudFront 콘솔에서 다음 경로를 Invalidation 대상으로 등록한다.

```text
/
/index.html
```

또는 실습 단계에서는 전체 경로를 무효화할 수도 있다.

```text
/*
```

단, `/*` 경로를 사용하면 모든 캐시 객체를 무효화하므로 실무에서는 필요한 경로만 지정하는 것이 비용과 관리 측면에서 더 좋다.

### Invalidation 생성 이유

S3에 최신 파일을 업로드해도 CloudFront Edge Location에는 이전 파일이 캐시되어 있을 수 있다.

Invalidation을 생성하면 CloudFront가 해당 경로의 캐시를 제거하고, 다음 요청 시 S3 Origin에서 최신 파일을 다시 가져온다.

정리하면 다음과 같다.

```text
S3에 index.html 재업로드
CloudFront에는 이전 index.html 캐시 존재 가능
Invalidation 생성
CloudFront 캐시 제거
다음 요청 시 최신 index.html 반환
```

Invalidation 상태가 `Completed`가 되면 캐시 무효화가 완료된 것이다.

## 10. S3 직접 접근 차단 확인

CloudFront를 통하지 않고 S3 객체 URL로 직접 접근했을 때 차단되는지 확인한다.

예시:

```text
https://버킷이름.s3.amazonaws.com/index.html
```

확인 결과:

```text
S3 직접 접근 시 AccessDenied 또는 403 Forbidden 발생
CloudFront 도메인으로 접근 시 정상 표시
```

이를 통해 S3 버킷은 직접 공개하지 않고, CloudFront를 통해서만 정적 사이트에 접근하도록 구성되었음을 확인했다.

## 11. S3 Versioning 확인

S3 Versioning이 활성화된 상태에서 `index.html`을 수정 후 재업로드하면 기존 파일이 덮어쓰기되는 것이 아니라 새로운 버전으로 저장된다.

확인 항목:

```text
index.html에 여러 Version ID가 생성됨
최신 버전이 현재 표시 대상이 됨
이전 버전도 보존됨
```

S3 Versioning을 통해 실수로 파일을 덮어쓰거나 삭제했을 때 이전 버전으로 복구할 수 있다.

이번 실습에서는 `index.html` 재업로드 후 새로운 버전이 생성되는 것을 확인했다.

## 12. 배운 점

이번 실습을 통해 다음 내용을 배웠다.

- CloudFront는 S3 Origin의 응답을 캐시한다.
- S3에 파일을 수정해서 업로드해도 CloudFront 캐시 때문에 이전 파일이 보일 수 있다.
- CloudFront Invalidation을 사용하면 캐시된 객체를 강제로 무효화할 수 있다.
- `/`와 `/index.html`은 같은 페이지처럼 보여도 CloudFront 캐시 관점에서는 다르게 동작할 수 있다.
- Default root object를 `index.html`로 설정하면 루트 경로 접근 시 `index.html`이 반환된다.
- S3를 비공개로 구성하면 존재하지 않는 객체 접근이 `404`가 아니라 `403`으로 처리될 수 있다.
- CloudFront Custom Error Response를 사용하면 `403` 응답을 사용자 정의 `404` 페이지로 변환할 수 있다.
- S3 직접 접근은 차단하고 CloudFront를 통해서만 접근하도록 구성할 수 있다.
- S3 Versioning을 사용하면 같은 객체의 이전 버전을 보존할 수 있다.

## 13. 다음 단계

다음 단계에서는 CloudFront 캐시 정책과 응답 헤더 정책을 더 자세히 확인한다.

예상 학습 내용:

- Cache Policy 기본 개념
- Origin Request Policy 개념
- Response Headers Policy 개념
- 정적 파일 캐시 전략
- HTML, CSS, JS 파일별 캐시 설정 차이
- CloudFront 보안 헤더 설정

