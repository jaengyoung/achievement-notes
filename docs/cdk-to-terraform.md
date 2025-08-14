# AWS CDK에서 Terraform으로: 버전 지옥 탈출과 상태 관리 개선 여정

## 🔁 마이그레이션 배경

### 실제 운영 경험: Java 버전 문제와 인프라 장애

기존 팀 프로젝트의 Infrastructure as Code(IaC)가 AWS CDK로 구성되어 있었으나, 다음과 같은 심각한 문제들로 인해 **Terraform으로의 전환을 결정**했습니다:

#### Java 버전 호환성 문제
- **Java 17 → Java 21 업그레이드** 시 CDK 호환성 이슈 발생
- CDK 버전별 요구 Java 버전이 상이하여 개발 환경 통일 어려움
- 팀 내 다른 Java 버전 사용으로 인한 빌드 실패

#### CloudFormation 배포 실패와 인프라 장애
- 버전 불일치로 인한 **CloudFormation 템플릿 생성 실패**
- 배포 중 스택이 중간 상태로 남아 **인프라 상태 꼬임**
- 복구를 위해 수동 CloudFormation 조작 필요
- **운영 환경 다운타임** 발생

#### 기타 운영상 문제들
- **복잡한 파일 관리**: 관리해야 할 파일이 과도하게 많음 
- **이관 프로젝트 이해 어려움**: 기존 CDK 코드의 복잡성으로 인한 학습 곡선
- **버전 관리 복잡성**: CDK CLI와 라이브러리 버전 불일치 문제
- **혼합 방식 적용시 문제**: 초기에는 콘솔 제어도 했기 때문에 CloudFormation이 변경 감지를 못하여 인프라 꼬인 경험 존재

이러한 문제들이 결국 **CDK에서 Terraform으로 마이그레이션**을 결정하게 된 주요 계기가 되었습니다.


## 🧐 발견한 CDK의 단점 3가지

### 첫 번째 단점: 인프라 상태 관리 주체는 사실 CloudFormation

#### CDK + CloudFormation: "건축설계사 + 시공업체" 모델

**역할 분담**
- **CDK (건축설계사)**: 고급 설계 도구로 편리하게 설계도 작성
- **CloudFormation (시공업체)**: 설계도를 받아서 실제 건물 시공

**작업 과정**
```
1. 건축설계사(CDK)가 고급 CAD로 설계
2. 설계도를 시공업체(CloudFormation) 형식으로 변환
3. 시공업체가 설계도 보고 건물 시공
4. 시공 중 문제 발생 시 설계사와 시공업체 사이 소통 필요
```

**현실적 문제들**
- **의사소통 문제**: 설계사가 의도한 것과 시공업체가 이해한 것이 다를 수 있음
- **책임 소재**: 문제 발생 시 설계사 탓인지, 시공업체 탓인지 애매함
- **변경 감지 한계**: 시공업체가 "설계도대로 했다"고 하지만 실제 건물 상태는 다를 수 있음

#### Terraform: "만능 건축가" 모델

**통합된 역할**
- **Terraform**: 설계부터 시공, 관리까지 모든 것을 직접 담당하는 만능 건축가

**작업 과정**
```
1. 건축가가 직접 설계하고 시공
2. 건물 상태를 실시간으로 파악
3. 변경사항 발생 시 즉시 감지
4. 문제 해결도 직접 담당
```

**장점들**
- **완전한 책임**: 한 사람이 모든 것을 담당하므로 책임이 명확
- **정확한 상태 파악**: 건축가가 직접 지었으므로 건물 상태를 정확히 알고 있음
- **빠른 대응**: 중간 단계 없이 바로 수정 가능

**실제 경험 사례로 본 차이점**

**Case 1. 리모델링 상황**

CDK+CloudFormation 방식:
```
건축설계사(CDK): "제가 Java 17로 설계했는데..."
시공업체(CloudFormation): "저희는 Java 21 기준으로 시공했습니다"
건축설계사: "그럼 설계도를 다시 그려야..."
시공업체: "그런데 이미 반쯤 지어서 철거가 어렵습니다"
결과: 집이 반쯤 부서진 상태로 방치 (인프라 장애)
```

Terraform 방식:
```
만능 건축가(Terraform): "현재 상태 확인... 변경 필요한 부분 파악... 바로 수정 완료!"
- 중간 과정 없이 직접 처리
- 실제 건물 상태와 설계가 항상 일치
- 문제 발생 시 즉시 원인 파악 및 해결
```

**Case 2. 상태 변경 감지 비유: "무단 개조 발견"**

CDK+CloudFormation 방식:
```
건축설계사: "설계도상으로는 문제없어 보이는데요?"
시공업체: "저희도 설계도대로 했습니다"
실제 상황: 누군가 몰래 벽에 구멍을 뚫고 새 문을 만들었음
결과: 무단 개조를 발견하지 못하고 계속 방치
```

Terraform 방식:
```
만능 건축가: "어? 벽에 새 문이 생겼네요!"
"누가 언제 만들었는지, 어떻게 원래대로 복구할지 다 알고 있습니다"
"바로 수정할까요? 아니면 이 상태로 승인할까요?"
```

**비유 정리표**

| 측면 | CDK+CloudFormation | Terraform |
|------|-------------------|-----------|
| **인력 구조** | 설계사 + 시공업체 (분업) | 만능 건축가 (통합) |
| **의사소통** | 복잡 (중간 단계 필요) | 단순 (직접 소통) |
| **변경 감지** | 제한적 (설계도 기준) | 완벽 (실제 상태 기준) |

### 두 번째 단점: 인프라 상태 변경 감지 능력 차이

실제 두 도구의 **인프라 상태 변경 감지 능력**을 실험적으로 비교 분석한 결과, 명확한 차이를 확인할 수 있었습니다.

**실험 환경**
- **AWS 리전**: ap-northeast-2 (서울)
- **AWS 계정**: 289023186990
- **AWS 프로필**: skale
- **CDK 버전**: 2.100.0 (Python)
- **Terraform 버전**: 1.x (AWS Provider 5.100.0)
- **테스트 리소스**: S3 버킷

**배포된 리소스**

CDK Python 프로젝트:
```
버킷 이름: cdk-experiment-bucket-r66zafdo
버킷 ARN: arn:aws:s3:::cdk-experiment-bucket-r66zafdo
태그: Environment=test, Project=simple-experiment, Tool=CDK-Python
```

Terraform 프로젝트:
```
버킷 이름: terraform-experiment-bucket-w88wn2dh
버킷 ARN: arn:aws:s3:::terraform-experiment-bucket-w88wn2dh
태그: Environment=test, Project=simple-experiment, Tool=Terraform
```

**수동으로 적용한 변경사항**
AWS 콘솔에서 다음 변경사항을 양쪽 버킷에 모두 적용:

1. **기존 태그 값 수정**:
   - `Environment`: `test` → `production`
   - `Project`: `simple-experiment` → `drift-test`

2. **새 태그 추가**:
   - `Owner: manual-change`
   - `LastModified: 2024-01-15`

3. **S3 설정 변경**:
   - 버전 관리: 비활성화 → 활성화

#### 실험 결과 비교

**1. CDK 상태 감지 테스트**

실행 명령어:
```bash
cd experiment/cdk-python
.venv/Scripts/activate
cdk diff --profile skale
```

실제 출력 로그:
```
start: Building 331def7ea7be60f7d079fc39da64d58a223bcb4d7277bf2b310c134e2d6ca2cb:289023186990-ap-northeast-2
success: Built 331def7ea7be60f7d079fc39da64d58a223bcb4d7277bf2b310c134e2d6ca2cb:289023186990-ap-northeast-2
start: Publishing 331def7ea7be60f7d079fc39da64d58a223bcb4d7277bf2b310c134e2d6ca2cb:289023186990-ap-northeast-2
success: Published 331def7ea7be60f7d079fc39da64d58a223bcb4d7277bf2b310c134e2d6ca2cb:289023186990-ap-northeast-2
Hold on while we create a read-only change set to get a diff with accurate replacement information (use --no-change-set to use a less accurate but faster template-only diff)
Stack S3BucketStack
Resources
[~] AWS::S3::Bucket ExperimentBucket ExperimentBucket20031DA2 replace
 └─ [~] BucketName (requires replacement)
     ├─ [-] cdk-experiment-bucket-r66zafdo
     └─ [+] cdk-experiment-bucket-wnjybp3n

✨  Number of stacks with differences: 1
```

CDK 결과 분석:
- **감지된 변경사항**: 버킷 이름 변경만 감지 (랜덤 이름 재생성)
- **놓친 변경사항**: 
  - ❌ 태그 값 변경 (`Environment`, `Project`)
  - ❌ 새로 추가된 태그 (`Owner`, `LastModified`)
  - ❌ 버전 관리 설정 변경
- **감지 정확도**: **매우 낮음** - 실제 드리프트 대부분 놓침

**2. Terraform 상태 감지 테스트**

실행 명령어:
```bash
cd experiment/terraform
terraform plan -detailed-exitcode
```

실제 출력 로그:
```
random_string.bucket_suffix: Refreshing state... [id=w88wn2dh]
aws_s3_bucket.experiment_bucket: Refreshing state... [id=terraform-experiment-bucket-w88wn2dh]
aws_s3_bucket_versioning.experiment_bucket_versioning: Refreshing state... [id=terraform-experiment-bucket-w88wn2dh]
aws_s3_bucket_public_access_block.experiment_bucket_pab: Refreshing state... [id=terraform-experiment-bucket-w88wn2dh]

Terraform used the selected providers to generate the following execution
plan. Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform planned the following actions, but then encountered a problem:

  # aws_s3_bucket.experiment_bucket will be updated in-place
  ~ resource "aws_s3_bucket" "experiment_bucket" {
        id                          = "terraform-experiment-bucket-w88wn2dh"
      ~ tags                        = {
          ~ "Environment"  = "production" -> "test"
          - "LastModified" = "2024-01-15" -> null
          - "Owner"        = "manual-change" -> null
          ~ "Project"      = "drift-test" -> "simple-experiment"
            "Tool"         = "Terraform"
        }
      ~ tags_all                    = {
          ~ "Environment"  = "production" -> "test"
          - "LastModified" = "2024-01-15" -> null
          - "Owner"        = "manual-change" -> null
          ~ "Project"      = "drift-test" -> "simple-experiment"
            # (1 unchanged element hidden)
        }
        # (12 unchanged attributes hidden)

        # (5 unchanged blocks hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.

Error: versioning_configuration.status cannot be updated from 'Enabled' to 'Disabled'

  with aws_s3_bucket_versioning.experiment_bucket_versioning,
  on main.tf line 39, in resource "aws_s3_bucket_versioning" "experiment_bucket_versioning":
  39: resource "aws_s3_bucket_versioning" "experiment_bucket_versioning" {
```

Terraform 결과 분석:
- **완벽하게 감지된 변경사항**:
  - ✅ `Environment` 태그: `"production" -> "test"`
  - ✅ `Project` 태그: `"drift-test" -> "simple-experiment"`
  - ✅ 추가된 태그 제거 예정: `Owner`, `LastModified`
  - ✅ 버전 관리 설정 충돌까지 감지 및 에러 표시
- **감지 정확도**: **완벽** - 모든 변경사항 100% 감지

**감지 능력 비교표**

| 변경 유형 | CDK `cdk diff` | Terraform `terraform plan` |
|-----------|----------------|--------------------------|
| 태그 값 수정 | ❌ 미감지 | ✅ 완벽 감지 |
| 새 태그 추가/삭제 | ❌ 미감지 | ✅ 완벽 감지 |
| 리소스 설정 변경 | ❌ 미감지 | ✅ 완벽 감지 + 에러 예측 |
| 출력 상세도 | 매우 제한적 | 매우 상세함 |
| 실행 속도 | 느림 (빌드 필요) | 빠름 (즉시 실행) |

**기술적 차이점**:
- **Terraform**: 실제 AWS 상태와 state 파일 비교
- **CDK**: 코드 기반 CloudFormation 템플릿 비교

**결론**: Terraform의 `terraform plan`이 CDK의 `cdk diff`보다 현저히 우수한 상태 변경 감지 능력을 보여줌

### 세 번째 단점: 파일 관리 및 버전 관리 복잡도 존재

#### CDK 파일 관리의 복잡성

**관리가 필요한 파일들**
```
✅ 버전 관리 필수:
   - cdk.json (CDK 설정)
   - requirements.txt (Python 의존성)
   - app.py, s3_bucket_stack.py (소스 코드)

❌ 버전 관리 제외:
   - cdk.out/ (빌드 결과물)
   - .venv/ (Python 가상환경)
   - __pycache__/ (Python 캐시)
```

**실험에서 관찰된 CDK 버전 문제**
실험 과정에서 다음과 같은 CDK 버전 관리 이슈가 확인되었습니다:

```
NOTICES (What's this? https://github.com/aws/aws-cdk/wiki/CLI-Notices)

34892   CDK CLI will collect telemetry data...
32775   (cli): CLI versions and CDK library versions have diverged
```

**문제점**:
- CDK CLI 버전과 라이브러리 버전이 분리되어 지속적인 경고 발생
- 매번 15-20초의 빌드 시간 필요 (`✨ Synthesis time: 15-20s`)
- 부트스트랩 버킷 생성 실패로 인한 배포 중단

#### Terraform 파일 관리의 단순성

**관리가 필요한 파일들**
```
✅ 버전 관리 필수:
   - main.tf, variables.tf, outputs.tf (인프라 정의)
   - .terraform.lock.hcl (프로바이더 버전 잠금)

❌ 버전 관리 제외:
   - terraform.tfstate (상태 파일 - 원격 백엔드 권장)
   - .terraform/ (프로바이더 캐시)
   - tfplan (임시 플랜 파일)
```

**장점**:
- 빌드 과정 없이 즉시 실행
- 자동 버전 잠금 (`.terraform.lock.hcl`)
- 단일 도구 의존성

**파일 관리 및 버전 관리 비교표**

| 측면 | CDK | Terraform |
|------|-----|-----------|
| **필수 관리 파일** | 4+ (다중 언어) | 4 (단일 도구) |
| **임시 파일 생성** | 많음 (cdk.out/, 빌드) | 적음 (.terraform/, state) |
| **버전 의존성** | 복잡 (CLI≠라이브러리, Java) | 단순 (CLI=프로바이더) |
| **빌드 과정** | 필요 (15-20초) | 불필요 (즉시 실행) |
| **버전 잠금** | 수동 관리 | 자동 (.terraform.lock.hcl) |
| **장애 복구** | 복잡 (CF 수동 조작) | 단순 (state 관리) |

## 📜 전환 결정 및 성과

### 마이그레이션 투자 시간과 성과
약 **5일간의 집중적인 학습과 마이그레이션 작업**을 통해 다음과 같은 성과를 달성했습니다:

- **100% 완전한 마이그레이션 완료**: 모든 인프라 리소스를 CDK에서 Terraform으로 성공적으로 전환
- **학습 효율성**: https://registry.terraform.io/providers/hashicorp/aws/latest/docs 사이트의 상세한 문서가 빠른 학습에 큰 도움
- **운영 안정성 확보**: 버전 호환성 문제로부터 완전히 해방
- **관리 복잡도 감소**: 단순하고 직관적인 인프라 관리 체계 구축

### 전환 후 주요 개선사항

1. **버전 관리 안정성**
   - Java 버전 의존성 완전 제거
   - 자동 프로바이더 버전 잠금 (`.terraform.lock.hcl`)
   - 팀 내 개발 환경 통일 달성

2. **상태 관리 신뢰성**
   - 실제 AWS 리소스 상태와 100% 일치하는 드리프트 감지
   - 즉시 실행되는 상태 확인 (`terraform plan`)
   - 인프라 변경사항의 정확한 예측 및 제어

3. **운영 효율성**
   - 빌드 시간 제거 (15-20초 → 0초)
   - 파일 관리 부담 경감
   - 장애 복구 프로세스 단순화

### 성공 요인

**Terraform 공식 문서의 우수성**
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs 에서 제공하는 상세하고 체계적인 가이드
- AWS 프로바이더 각 리소스별 명확한 예제와 설명
- 마이그레이션 과정에서 필요한 모든 정보를 효율적으로 제공

**직관적인 도구 설계**
- 학습 곡선이 완만하여 빠른 숙련도 달성
- 코드 작성과 상태 관리의 일관성
- 에러 메시지의 명확성과 해결 방향 제시

### 장기적 관점에서의 가치

이번 마이그레이션은 단순한 도구 변경을 넘어서 **인프라 운영 패러다임의 전환**을 의미합니다:

- **예방적 인프라 관리**: 문제 발생 전 사전 감지 및 대응
- **운영 리스크 최소화**: 버전 호환성 이슈로 인한 서비스 중단 방지
- **팀 생산성 향상**: 복잡한 빌드 과정과 디버깅 시간 단축
- **기술 부채 해소**: 지속 가능한 인프라 관리 체계 확립

결과적으로 **5일의 투자**로 **장기적인 운영 안정성과 효율성**을 확보하는 매우 성공적인 전환이었습니다.

---

**마이그레이션 수행일**: 2025-02-22 ~ 2025-02-28  
**실험 환경**: AWS ap-northeast-2 리전  
**사용 도구**: CDK Python 2.100.0, Terraform AWS Provider 5.100.0  
**투자 시간**: 약 5일 (학습 + 마이그레이션)  
**성과**: 100% 완전 전환 완료, 운영 안정성 확보