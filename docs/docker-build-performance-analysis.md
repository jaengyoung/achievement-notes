# Docker 빌드 성능 최적화: 캐시 전략으로 85% 빌드 시간 단축하기

## 🚀 도입

### 문제 배경
GitHub Actions를 사용해서 CI/CD를 구현하여 AWS Lambda를 배포하고 있는데, **컨테이너 이미지 기반 Lambda 배포 시 빌드 시간이 꽤 많이 소요**되는 문제가 있었습니다. 

특히 Go Lambda 애플리케이션의 경우, **조금만 수정해도 빌드 시간이 1분이 넘어가니까** 개발 과정에서 상당한 스트레스를 받게 되었습니다. 코드 변경 → 푸시 → 빌드 대기 → 배포 확인의 사이클에서 빌드 대기 시간이 개발 리듬을 크게 방해하고 있었죠.

더 최적화할 수 있는 방법은 없을지 고민하다가 **Docker Layer 캐싱을 활용한 빌드 시간 최적화**를 시도하게 되었습니다.

### 공유 목적
GitHub Actions CI/CD 환경에서 **컨테이너 이미지 기반 Lambda 배포**의 빌드 시간을 **85% 단축**한 실제 경험을 공유합니다. 비슷한 고민을 하고 있는 개발팀에게 바로 적용 가능한 실용적인 해결책을 제시하고자 합니다.

### 실제 적용 환경
- **배포 대상**: AWS Lambda (컨테이너 이미지 방식)
- **CI/CD 플랫폼**: GitHub Actions + ECR
- **언어/런타임**: Go 1.24
- **테스트 시나리오**: 실제 개발 과정을 모방한 4회 커밋
  1. 최초 커밋 (기본 Lambda 코드)
  2. 주석 추가 (코드 변경 최소)
  3. 가벼운 의존성 추가 (`github.com/tidwall/gjson`)
  4. 무거운 의존성 추가 (`github.com/gin-gonic/gin`)

---

## 📋 본문

### 시도한 방법

#### 1. Dockerfile 최적화

**기존 방법의 문제점:**
```dockerfile
# 문제: 전체 소스를 한 번에 복사
COPY .. .
RUN go mod download
```
- 소스 코드 한 줄만 변경되어도 의존성 다운로드부터 다시 실행
- Docker Layer 캐싱이 제대로 작동하지 않음

**개선 방법:**
```dockerfile
# 1단계: 의존성 파일만 먼저 복사
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# 2단계: 소스 코드는 나중에 복사
COPY main.go ./
COPY internal/ ./internal/
```

**핵심 전략:**
1. **레이어 분리**: 변경 빈도가 다른 파일들을 다른 레이어에 배치
2. **베이스 이미지 경량화**: `golang:1.24` → `golang:1.24-alpine`
3. **빌드 최적화**: 프로덕션용 최적화 플래그 추가

#### 2. GitHub Actions 워크플로우 개선

**기존 방법:**
```yaml
# 개별 Docker 명령어 사용 - 캐싱 없음
- name: Build the Docker image
  run: docker build -t "$ECR_REPOSITORY_NAME:$IMAGE_TAG" .
- name: Tag the Docker image  
  run: docker tag ...
- name: Push the Docker image
  run: docker push ...
```

**개선 방법:**
```yaml
# Docker Buildx + GitHub Actions 캐싱 활용
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Build and push to ECR
  uses: docker/build-push-action@v6
  with:
    cache-from: type=gha    # 캐시에서 읽기
    cache-to: type=gha,mode=min  # 캐시에 저장
```

**핵심 전략:**
1. **통합 액션 사용**: 3개 명령어 → 1개 액션으로 통합
2. **GitHub Actions 캐싱**: `type=gha`로 빌드 레이어 캐시
3. **최신 도구 활용**: Docker Buildx로 고급 캐싱 기능 사용

#### 3. 통합 최적화 효과

| 최적화 영역 | 개선 내용 | 예상 효과 |
|-------------|-----------|-----------|
| **Dockerfile** | 의존성/소스 분리 | 의존성 변경 없을 시 캐시 활용 |
| **베이스 이미지** | Alpine 사용 | 다운로드 시간 단축 |
| **빌드 도구** | Docker Buildx | 고급 캐싱 지원 |
| **CI/CD** | GitHub Actions 캐시 | 빌드 레이어 재사용 |

### 실험 결과

#### 성능 측정 결과

| 커밋 | 기존 방법 | 개선 방법 | 성능 향상 | 배속 개선 |
|------|-----------|-----------|-----------|-----------|
| 1회 (최초) | 39초 | 77초 | -97.4% | 0.5배 |
| 2회 (주석) | 39초 | 6초 | **84.6%** | **6.5배** |
| 3회 (gjson) | 40초 | 6초 | **85.0%** | **6.7배** |
| 4회 (gin) | 43초 | 6초 | **86.0%** | **7.2배** |

#### 핵심 성과 지표
- **반복 빌드 성능 향상**: **85.3%** (40.7초 → 6초)
- **속도 개선**: **6.8배**
- **의존성 확장성**: Gin 프레임워크 추가에도 동일한 6초 성능 유지

### 실패와 해결 과정

#### 1. 1회 커밋 성능 역전 현상
**문제**: 초기 빌드에서 개선 방법이 97% 더 느림 (39초 → 77초)

**원인 분석:**
| 단계 | 기존 방법 | 개선 방법 | 차이 | 원인 |
|------|-----------|-----------|------|------|
| 베이스 이미지 | 16.8초 | 17.7초 | +0.9초 | Alpine 설정 |
| 의존성 다운로드 | 3.5초 | 4.6초 | +1.1초 | verify 추가 |
| 빌드 과정 | 14.9초 | 34.9초 | +20초 | 최적화 플래그 |
| 캐시 설정 | 0초 | 15초 | +15초 | 초기 캐시 생성 |

**해결책**: Setup Cost vs Long-term Efficiency 관점에서 접근
- 1회는 초기 투자 비용으로 인정
- 2회부터 즉시 ROI 달성으로 장기적 효율성 확보

#### 2. 의존성 추가 시 캐시 무효화 우려
**우려**: 새로운 라이브러리 추가 시 캐시가 깨질 것

**실제 결과**: GitHub Actions의 intelligent caching으로 해결
- 3회 커밋: gjson 추가 → 6초 유지
- 4회 커밋: gin 추가 → 6초 유지
- **의존성 크기와 무관하게 일정한 성능**

### 코드 구현 세부사항

#### 1. Dockerfile 차이점

**기존 방법:**
```dockerfile
# -------- Stage 1: Builder --------
FROM golang:1.24 AS builder
WORKDIR /app

# 문제: 의존성과 소스를 함께 복사
COPY go.mod go.sum ./
RUN go mod download

# 전체 소스 파일 복사 - 캐시 무효화 원인
COPY .. .

# 기본 빌드 (최적화 없음)
RUN GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o main main.go

# -------- Stage 2: Lambda Runner --------
FROM public.ecr.aws/lambda/go:1
COPY --from=builder /app/main ${LAMBDA_TASK_ROOT}
CMD ["main"]
```

**개선 방법:**
```dockerfile
# -------- Stage 1: Builder (경량 이미지) --------
FROM golang:1.24-alpine AS builder
WORKDIR /app

# 핵심: 종속성 파일만 먼저 복사 (Docker 레이어 캐싱 최적화)
COPY go.mod go.sum ./
# 종속성 다운로드 + 검증
RUN go mod download && go mod verify

# 소스 코드는 나중에 복사 (필요한 파일만)
COPY main.go ./
COPY internal/ ./internal/

# Lambda 컨테이너용 정적 빌드 (최적화 플래그)
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags="-s -w -extldflags '-static'" \
    -a -installsuffix cgo \
    -trimpath \
    -o main main.go

# -------- Stage 2: Lambda Go Runtime --------
FROM public.ecr.aws/lambda/go:1
COPY --from=builder /app/main ${LAMBDA_TASK_ROOT}/main
CMD ["main"]
```

#### 2. GitHub Actions 워크플로우 차이점

**기존 방법:**
```yaml
- name: Checkout the repository
  uses: actions/checkout@v3  # 구버전

# 개별 Docker 명령어 사용
- name: Build the Docker image
  run: |
    docker build -t "$ECR_REPOSITORY_NAME:$IMAGE_TAG" .

- name: Tag the Docker image
  run: |
    docker tag "$ECR_REPOSITORY_NAME:$IMAGE_TAG" \
      "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY_NAME:$IMAGE_TAG"

- name: Push the Docker image to ECR
  run: |
    docker push "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY_NAME:$IMAGE_TAG"
```

**개선 방법:**
```yaml
- name: Checkout the repository
  uses: actions/checkout@v4  # 최신 버전

# 핵심: Docker Buildx 설정으로 캐싱 지원
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

# 통합된 빌드+푸시 액션 + GitHub Actions 캐싱
- name: Build and push to ECR
  uses: docker/build-push-action@v6
  with:
    context: source/lambda
    push: true
    tags: ${{ env.ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY_NAME }}:${{ env.IMAGE_TAG }}
    platforms: linux/amd64
    cache-from: type=gha    # GitHub Actions 캐시에서 읽기
    cache-to: type=gha,mode=min  # GitHub Actions 캐시에 저장
```

#### 3. 핵심 차이점 요약

| 구분 | 기존 방법 | 개선 방법 | 개선 효과 |
|------|-----------|-----------|-----------|
| **베이스 이미지** | `golang:1.24` | `golang:1.24-alpine` | 경량화 |
| **파일 복사 전략** | `COPY .. .` (전체) | `COPY main.go ./` + `COPY internal/` | 레이어 분리 |
| **빌드 최적화** | 기본 빌드 | `-ldflags="-s -w"` 등 | 바이너리 최적화 |
| **의존성 검증** | `go mod download` | `go mod download && go mod verify` | 무결성 보장 |
| **GitHub Actions** | 개별 명령어 | `docker/build-push-action@v6` | 통합 액션 |
| **캐싱 전략** | 없음 | `cache-from/to: type=gha` | GitHub Actions 캐싱 |

---

## 🎯 정리

### 핵심 배운 점

#### 1. Docker Layer 캐싱의 위력
- **의존성과 소스 코드 분리**만으로도 85% 성능 향상 달성
- 변경 빈도가 다른 레이어를 전략적으로 배치하는 것이 핵심

#### 2. Setup Cost vs Long-term Efficiency 패러다임
- 초기 설정 비용이 높더라도 장기적 효율성을 고려한 투자 필요
- 2회 커밋부터 즉시 ROI 달성으로 빠른 회수 가능

#### 3. GitHub Actions 캐싱의 고도화
- intelligent caching으로 의존성 추가에도 캐시 효과 유지
- Multi-layer 캐싱 전략의 효과성 입증

### 다른 상황 적용 팁

#### 1. 언어별 최적화 전략
```dockerfile
# Node.js
COPY package*.json ./
RUN npm ci --only=production

# Python
COPY requirements.txt ./
RUN pip install -r requirements.txt

# Java
COPY pom.xml ./
RUN mvn dependency:go-offline
```

#### 2. 캐시 친화적 Dockerfile 작성 원칙
1. **변경 빈도가 낮은 것부터**: 의존성 → 설정 → 소스 코드
2. **경량 베이스 이미지**: Alpine 계열 우선 고려
3. **Multi-stage build**: 빌드 환경과 런타임 환경 분리
4. **빌드 최적화**: 프로덕션용 최적화 플래그 적용

#### 3. CI/CD 파이프라인 최적화
- **캐시 키 전략**: 의존성 파일 해시 기반 캐시 키 설정
- **병렬 빌드**: 독립적인 컴포넌트는 병렬 실행
- **조건부 빌드**: 변경된 컴포넌트만 선택적 빌드

### 비즈니스 임팩트

#### 정량적 효과
- **개발자 시간 절약**: 월간 200회 빌드 기준 19.4시간 절약
- **인프라 비용**: GitHub Actions 실행 시간 85% 절감
- **배포 속도**: 6.8배 빠른 빌드로 개발 사이클 단축

#### 정성적 효과
- **개발자 경험 향상**: 대기 시간 최소화로 집중도 향상
- **CI/CD 안정성**: 일관된 빌드 성능으로 예측 가능성 증가
- **확장성**: 프로젝트 복잡도 증가에도 성능 유지

---

## 📚 참고자료

### 기술 문서
- [Docker Best Practices for Writing Dockerfiles](https://docs.docker.com/develop/dev-best-practices/)
- [GitHub Actions Caching Dependencies](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)
- [Go Build Optimization](https://golang.org/cmd/go/#hdr-Compile_packages_and_dependencies)

### 도구 및 플랫폼
- **Docker**: Multi-stage builds and layer caching
- **GitHub Actions**: Workflow caching strategies
- **Go**: Build optimization flags and module management

---

*실험 일시: 2025-08-12*  
*환경: GitHub Actions, AWS ECR, Go 1.24*