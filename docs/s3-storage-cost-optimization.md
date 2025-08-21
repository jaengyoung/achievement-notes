# S3 스토리지 비용 폭증 해결: 압축 알고리즘 개선으로 87% 용량 절감 달성

## 👻 요약

> 어떤 성과를 정리했습니다.
> 
> 어떤 성과를 내었습니다.
> 
> 이렇게 정리할까요?


## 🧐 시작하며

### 문제 배경
7월 말부터 '더보기' 검색 크롤링을 강화하면서 **S3 저장량이 급격히 증가**하기 시작했습니다. 매일 약 300만 개의 키워드를 크롤링하며 어느 정도 요금 증가를 예상했지만, **예상을 훨씬 뛰어넘는 폭발적인 비용 증가**가 발생했습니다. 특히 S3 청구 요금이 **$54.73 → $60.39 → $180.87로 230% 급증**하는 상황에서, 실제로 하루에 약 30GB씩 저장량이 늘어나는 심각한 양상을 보였습니다. 기존 PARQUET (Snappy) 압축 방식으로는 **파일당 평균 9.22KB의 크기**를 기록하며, 대용량 크롤링 데이터에 대한 근본적인 저장 전략 재검토가 필요한 상황이었습니다.

### 공유 목적
S3 스토리지 비용 폭증 상황에서 **압축 알고리즘 개선을 통해 87% 용량 절감**을 달성한 실제 경험을 공유합니다. 파일 포맷과 압축 알고리즘의 차이를 명확히 이해하고, 데이터 특성에 맞는 최적 압축 전략 선택의 중요성을 제시하고자 합니다.

<!--
### 실제 적용 환경
- **크롤링 규모**: 일일 300만 개 키워드 데이터
- **데이터 증가량**: 하루 30GB씩 지속적 증가
- **기존 방식**: PARQUET (Snappy) 압축
- **데이터 특성**: 텍스트 중심, 반복 패턴 많음
- **비용 임팩트**: S3 요금 230% 급증 ($54.73 → $180.87)
-->

##  🔥 해결 과정

### 기존 저장 방식의 한계 분석

#### 1️⃣ PARQUET (Snappy) 방식의 문제점

```python
import pandas as pd
df.to_parquet('crawl_data.parquet', compression='snappy')
```

- **압축 알고리즘 목적의 차이** : 속도 우선, 압축률 희생
- **컬럼 기반 구조 오버헤드** : 메타데이터 포함으로 작은 파일에서 비효율
- **텍스트 데이터 특성 미반영** : 반복 패턴 인식 약함

#### 2️⃣ 저장 비용 폭증 분석

| 날짜 | 누적 사용량 (GB) | 일일 증가량 (GB) |
|------|------------------|------------------|
| July 24, 2025 | 61.5 | - |
| July 25, 2025 | 93.2 | 31.7 |
| July 26, 2025 | 117 | 23.8 |
| July 27, 2025 | 147 | 30.0 |
| July 28, 2025 | 175 | 28.0 |
| July 29, 2025 | 203 | 28.0 |
| July 30, 2025 | 232 | 29.0 |
| July 31, 2025 | 257 | 25.0 |
| August 1, 2025 | 285 | 28.0 |
| August 2, 2025 | 313 | 28.0 |
| August 3, 2025 | 342 | 29.0 |
| August 4, 2025 | 367 | 25.0 |

- **1단계**: $54.73 (기준)
- **2단계**: $60.39 (+10.3%)
- **3단계**: $180.87 (+230.4%) ← **임계점 도달**

### 파일 포맷 vs 압축 알고리즘 분석

#### 1️⃣ 파일 포맷 vs 압축 알고리즘 비교

| 구분 | 의미 | 예시 | 압축률에 영향 |
| --- | --- | --- | --- |
| **파일 포맷** | 데이터를 어떻게 저장할지 결정 | CSV, PARQUET, ORC, JSON | 약간 영향 |
| **압축 알고리즘** | 데이터를 얼마나 작게 압축할지 결정 | GZIP, Snappy, ZSTD, Brotli | **가장 큰 영향** |

#### 2️⃣ 파일 포맷 종류 분석

| 항목 | CSV | PARQUET |
| --- | --- | --- |
| **저장 방식** | row-based | columnar (컬럼 기반) |
| **스키마 포함 여부** | 없음 | 있음 (메타데이터 포함) |
| **효율적 쿼리** | ❌ 전체 스캔 필요 | ✅ 컬럼 단위 스캔 가능 |
| **포맷 자체 압축** | ❌ 없음 | ✅ 내부 블록 압축 지원 |
| **압축률** | 포맷 자체 영향은 적음 | 포맷 + 압축 방식 복합 영향 |

#### 3️⃣ 압축 알고리즘 종류 분석

| 항목 | Snappy | GZIP | ZSTD |
| --- | --- | --- | --- |
| **목적** | 속도 중심 | 압축률 중심 | 속도 + 압축률 |
| **압축률** | 중간 (빠르지만 낮음) | 높음 | 매우 높음 |
| **속도** | ⚡ 매우 빠름 | ⏳ 느림 | ⚡ 빠름 (GZIP보다 빠르고 작음) |

#### 4️⃣ 실제 조합 예시

- `CSV.GZ` : CSV 텍스트를 **GZIP**으로 압축
- `PARQUET + Snappy` : 바이너리 포맷을 **Snappy**로 압축  
- `PARQUET + GZIP` or `ZSTD` : 포맷은 같고, **압축 알고리즘만 변경**

### 단일 실험 설계 및 결과

#### 1️⃣ 실험 설계

```python

import pandas as pd
import gzip
import os

# 테스트 데이터 생성
test_data = generate_crawl_data(keywords=20, ranks=10)  # 200 records

# 방법 1: Parquet (Snappy)
test_data.to_parquet('test_data.parquet', compression='snappy')
parquet_size = os.path.getsize('test_data.parquet')

# 방법 2: CSV + GZIP
test_data.to_csv('test_data.csv', index=False)
with open('test_data.csv', 'rb') as f_in:
    with gzip.open('test_data.csv.gz', 'wb') as f_out:
        f_out.writelines(f_in)
csvgz_size = os.path.getsize('test_data.csv.gz')
```

- **실험 데이터** : 20개 키워드 1~10위까지 크롤링 데이터
- 총 200개 레코드를 저장하여 크기 비교 

#### 2️⃣ 실험 결과

```
🚀 압축률 테스트 시작...
📊 생성된 데이터: 200개 레코드

📦 PARQUET 파일 저장 중...
✅ PARQUET 파일 크기: 12,512 bytes (12.22 KB)

📦 CSV.GZ 파일 저장 중...
✅ CSV.GZ 파일 크기: 1,980 bytes (1.93 KB)

📈 압축률 비교 결과
PARQUET: 12.22 KB
CSV.GZ:   1.93 KB
-> CSV.GZ가 84.18% 더 효율적입니다!
```

#### 3️⃣ 압축 매커니즘 차이 분석

```python
# 크롤링 데이터의 특징
sample_data = {
    'keyword': ['노트북', '노트북', '노트북', ...],  # 반복 패턴
    'rank': [1, 2, 3, 4, 5, ...],                   # 순차 패턴  
    'title': ['삼성 노트북...', 'LG 노트북...'],     # 유사 패턴
    'url': ['https://..', 'https://..'],            # 공통 접두사
}

# GZIP의 장점: 이러한 반복/유사 패턴을 효과적으로 압축
```

- **GZIP** : LZ77 알고리즘으로 반복 문자열 참조
- **Snappy** : 빠른 압축/해제 우선, 압축률 제한적
- **텍스트 데이터** : 반복 패턴이 많아 GZIP에 최적화

### 프로덕션 환경 적용

#### 1️⃣ 코드 개선

**기존 방식 (PARQUET + SNAPPY)**
```go
// 문제: 복잡한 구조체와 Snappy 압축의 한계
type ParquetRow struct {
    Query       string `parquet:"name=query, type=BYTE_ARRAY, convertedtype=UTF8, encoding=PLAIN_DICTIONARY"`
    Device      string `parquet:"name=device, type=BYTE_ARRAY, convertedtype=UTF8, encoding=PLAIN_DICTIONARY"`
    Rank        string `parquet:"name=rank, type=BYTE_ARRAY, convertedtype=UTF8, encoding=PLAIN_DICTIONARY"`
    SiteName    string `parquet:"name=site_name, type=BYTE_ARRAY, convertedtype=UTF8, encoding=PLAIN_DICTIONARY"`
    DisplayURL  string `parquet:"name=display_url, type=BYTE_ARRAY, convertedtype=UTF8, encoding=PLAIN_DICTIONARY"`
    Title       string `parquet:"name=title, type=BYTE_ARRAY, convertedtype=UTF8, encoding=PLAIN_DICTIONARY"`
    Description string `parquet:"name=description, type=BYTE_ARRAY, convertedtype=UTF8, encoding=PLAIN_DICTIONARY"`
}

// Parquet 파일 생성 (복잡한 설정)
pw, err := writer.NewParquetWriter(buffer, new(ParquetRow), 4)
pw.RowGroupSize = 128 * 1024 * 1024
pw.CompressionType = parquet.CompressionCodec_SNAPPY
```

**개선 방식 (CSV + GZIP)**
```go
// 해결: 단순한 구조체와 효율적인 GZIP 압축
type CrawlingResult struct {
    Query       string `json:"query"`
    Device      string `json:"device"`
    Rank        int    `json:"rank"`
    SiteName    string `json:"site_name"`
    DisplayURL  string `json:"display_url"`
    Title       string `json:"title"`
    Description string `json:"description"`
}

// CSV.GZ 파일 생성 (단순한 설정)
buffer := new(bytes.Buffer)
gzWriter := gzip.NewWriter(buffer)
csvWriter := csv.NewWriter(gzWriter)

// 데이터 스트리밍 처리
for _, item := range result {
    record := []string{
        item["query"].(string),
        item["device"].(string),
        strconv.Itoa(item["rank"].(int)),
        // ... 나머지 필드들
    }
    csvWriter.Write(record)
}
```

- **구조체 단순화** : 복잡한 PARQUET 태그 → 간단한 JSON 태그
- **압축 방식 변경** : Snappy → GZIP (87% 효율 향상)
- **스트리밍 처리** : 메모리 효율적인 파이프라인 구성
- **파일 구조** : 컬럼 기반 → 행 기반 단순 텍스트

#### 2️⃣ 적용 결과

**적용 전 (2025.08.05 09:00 KST)**
- **Total Objects** : 194,851개
- **Total Size** : 1,870,300,351 Bytes (1.74GB)
- **파일당 평균 크기** : 9.22KB

**적용 후 (2025.08.05 10:00 KST)**
- **Total Objects** : 195,545개  
- **Total Size** : 233,826,129 Bytes (0.22GB)
- **파일당 평균 크기**: 1.20KB

#### 3️⃣ 핵심 성과 지표

| 메트릭 | 적용 전 | 적용 후 | 개선 효과 |
|--------|---------|---------|-----------|
| **파일당 평균 크기** | 9.22 KB | 1.20 KB | **-87.0%** |
| **전체 저장 용량** | 1.74 GB | 0.22 GB | **-87.4%** |
| **압축률 향상** | 기준 (1배) | **7.68배** | 768% 향상 |
| **비용 절감률** | - | **87.49%** | 대폭 절약 |

- **즉시 효과** : 1.74GB → 0.22GB (87.4% 용량 절감)
- **일일 저장량** : 30GB → 3.8GB (87% 절약)
- **예상 월간 S3 비용** : $180.87 → $22.61 (87.5% 절감)
- **연간 절약 효과** : 약 $1,900 절약 가능

---

## 🎯 정리

### 핵심 배운 점

#### 1. 데이터 특성에 맞는 압축 방식 선택의 중요성
- **텍스트 중심 데이터**: 반복 패턴이 많은 경우 GZIP이 압도적으로 유리
- **구조적 오버헤드**: 작은 파일에서 Parquet의 메타데이터가 비효율 야기
- **용도별 최적화**: 분석용(Parquet) vs 저장용(CSV.GZ) 구분 필요

#### 2. 비용 최적화의 사전 예방적 접근
- **점진적 모니터링**: 일일 30GB 증가 패턴 조기 감지 중요
- **임계점 인식**: $180 수준에서 긴급 대응 필요성 판단
- **실험적 검증**: 소규모 테스트로 효과 사전 검증

#### 3. 압축 알고리즘의 트레이드오프 이해
- **GZIP**: 높은 압축률, 느린 속도 (저장 중심)
- **Snappy**: 빠른 속도, 낮은 압축률 (처리 중심)
- **상황별 선택**: 비용 vs 성능의 균형점 찾기

### 다른 상황 적용 팁

#### 1. 대용량 텍스트 데이터 저장 전략
```python
# 데이터 특성별 압축 방식 선택 가이드
def choose_compression(data_characteristics):
    if data_characteristics['text_heavy'] and data_characteristics['repetitive']:
        return 'gzip'  # 84% 압축 효율
    elif data_characteristics['analytical_queries']:
        return 'parquet'  # 빠른 컬럼 액세스
    else:
        return 'snappy'  # 범용적 선택
```

#### 2. 비용 모니터링 자동화
- **일일 사용량 추적**: CloudWatch로 S3 메트릭 모니터링
- **임계값 알람**: 30GB/일 초과 시 자동 알람
- **압축률 측정**: 정기적인 압축 효율성 검증

#### 3. 점진적 마이그레이션 전략
- **A/B 테스트**: 일부 데이터로 압축 방식 검증
- **백워드 호환성**: 기존 Parquet 데이터 유지 고려
- **배치 처리**: 대용량 데이터 변환 시 분할 처리

### 비즈니스 임팩트

#### 정량적 효과
- **파일당 크기 개선**: 9.22KB → 1.20KB (87% 절약)
- **전체 용량 절감**: 1.74GB → 0.22GB (87.4% 절감)
- **실제 비용 절감**: 월 $158.26 절약 (87.5% 절감)
- **연간 절약 효과**: 약 $1,900 비용 절감 달성

#### 정성적 효과
- **확장성 확보**: 크롤링 규모 확대에도 비용 통제 가능
- **운영 안정성**: 예측 가능한 저장 비용으로 예산 계획 수립
- **기술적 인사이트**: 데이터 특성 기반 최적화 노하우 축적

---

## 📚 참고자료

### 기술 문서
- [AWS S3 Storage Classes and Pricing](https://aws.amazon.com/s3/pricing/)
- [Pandas Compression Options](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.to_csv.html)
- [GZIP vs Snappy Compression Comparison](https://github.com/google/snappy)

### 압축 알고리즘 및 최적화
- **GZIP**: LZ77 기반 무손실 압축, 높은 압축률
- **Snappy**: Google 개발, 빠른 압축/해제 속도 중심
- **Storage Optimization**: 클라우드 비용 최적화 전략

---

*실험 일시: 2025-08-12*  
*환경: AWS S3, 일일 300만 키워드 크롤링 데이터*