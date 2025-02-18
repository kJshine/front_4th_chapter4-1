# Next.js AWS 배포 파이프라인

## 목차

- [아키텍처](#아키텍처)
- [배포 프로세스](#배포-프로세스)
- [인프라 구성 요소](#인프라-구성-요소)
- [GitHub Actions 설정](#GitHub-Actions-설정)
- [성능 최적화](#성능-최적화)
- [주요 링크](#주요-링크)
- [향후 개선사항](#향후-개선사항)

## 아키텍처

![AWS 배포 파이프라인 다이어그램](/app/assets/images/aws_deployment_pipeline.png)
_Next.js 프로젝트의 AWS 배포 파이프라인 구성도_

### 주요 구성 요소

1. **GitHub Actions**: CI/CD 파이프라인
2. **Amazon S3**: 정적 웹 호스팅
3. **CloudFront**: CDN 서비스
4. **IAM**: 보안 및 권한 관리

## 배포 프로세스

1. 개발자가 main 브랜치에 푸시
2. GitHub Actions 자동 실행
3. `npm ci` 명령어로 프로젝트 의존성 설치
4. `npm run build` 명령어로 Next.js 프로젝트 빌드
5. AWS 인증 설정
6. S3에 빌드 결과물 업로드
7. CloudFront 캐시 무효화

## 인프라 구성 요소

### Amazon S3 버킷 설정

- 퍼블릭 액세스 차단 설정
- 버킷 정책 설정
- CORS 설정

```javascript
[
  {
    AllowedHeaders: ["*"],
    AllowedMethods: ["GET", "HEAD"],
    AllowedOrigins: ["*"],
    ExposeHeaders: [],
  },
];
```

프로덕션 환경에서는 도메인을 구입후 AllowedOrigins에 도메인을 추가해야 합니다.

- 정적 웹 사이트 호스팅 설정
  - 인덱스 문서: index.html
  - 오류 문서: 404.html

### Amazon CloudFront 설정

- 뷰어 프로토콜 정책 설정
  - HTTPS로 리다이렉션
- 웹 애플리케이션 방화벽 설정
  - WAF 비활성화

### AWS IAM 설정

- 사용자 커스텀 정책 생성
- 사용자 권한 설정

## GitHub Actions 설정

### 환경 변수 설정

| 환경 변수                    | 설명                       | 확인 방법                |
| ---------------------------- | -------------------------- | ------------------------ |
| `AWS_ACCESS_KEY_ID`          | IAM 계정의 액세스 키       | IAM 사용자 생성 시 발급  |
| `AWS_SECRET_ACCESS_KEY`      | IAM 계정의 비밀 액세스 키  | IAM 사용자 생성 시 발급  |
| `AWS_REGION`                 | S3 버킷이 위치한 리전 코드 | AWS 콘솔 우측 상단       |
| `S3_BUCKET_NAME`             | 배포 대상 S3 버킷 이름     | S3 콘솔에서 확인         |
| `CLOUDFRONT_DISTRIBUTION_ID` | CloudFront 배포 ID         | CloudFront 콘솔에서 확인 |

### 배포 워크플로우

저장소의 `.github/workflows/deployment.yml`에 다음 내용을 추가합니다:

```yaml
name: Deploy Next.js to S3 and invalidate CloudFront

on:
  push:
    branches:
      - main # 또는 master, 프로젝트의 기본 브랜치 이름에 맞게 조정
  workflow_dispatch: # 수동으로도 실행 가능

jobs: # 실행할 작업 정의
  deploy:
    runs-on: ubuntu-latest # Ubuntu 환경에서 실행

    steps: # 순차적으로 실행할 단계들
      - name: Checkout repository # 소스코드 가져오기
        uses: actions/checkout@v4

      - name: Install dependencies # npm 의존성 설치
        run: npm ci

      - name: Build # Next.js 프로젝트 빌드
        run: npm run build

      - name: Configure AWS credentials # AWS 인증 설정
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Deploy to S3 # S3에 빌드 결과물 업로드
        run: |
          aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }} --delete

      - name: Invalidate CloudFront cache # CloudFront 캐시 무효화
        run: |
          aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```

## 성능 최적화

1. CDN 활용

- S3 직접 액세스

![S3 성능](/app/assets/images/s3_performance.png)
_S3 직접 액세스 시 평균 응답 시간_

- CloudFront 사용

![CloudFront 성능](/app/assets/images/cloudfront_performance.png)
_CloudFront를 통한 콘텐츠 전송 시 응답 시간 개선_

## 주요 링크

- S3 버킷 웹사이트 엔드포인트: http://hanghae-website-assets.s3-website.us-east-2.amazonaws.com
- CloudFront 배포 도메인: https://d1tk4xx283d6nb.cloudfront.net/

## 향후 개선사항

- 도메인 구입 이후 Route 53 설정 및 도메인 연결
- 테스트 자동화 구축
