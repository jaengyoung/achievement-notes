# 확률 기반 트리 드리프팅: 키워드 분류 정확도 18.3% 향상과 처리 속도 3배 개선

## 🚀 도입

### 문제 배경
NOL 인터파크에서 사용자 검색 키워드를 적절한 카테고리로 자동 분류하는 시스템이 필요한 상황이었습니다. **기존 LLM + 프롬프트 엔지니어링 방식은 category_4에서 28.08%라는 사용 불가 수준의 정확도**를 보였고, 전통적 머신러닝 접근법도 계층적 구조를 제대로 반영하지 못하는 한계가 있었습니다.

특히 이커머스 플랫폼에서 검색 키워드의 정확한 카테고리 분류는 사용자 경험과 매출에 직결되는 핵심 기능이므로, **운영 가능한 수준의 성능과 안정적인 처리 속도**가 필수적이었죠.

더 효율적이면서도 정확한 방법은 없을지 고민하다가 **확률 기반 트리 드리프팅(Probabilistic Tree Drifting)을 활용한 계층적 키워드 분류**를 시도하게 되었습니다.

### 공유 목적
NOL 인터파크 키워드 분류 시스템에서 **정확도 18.3% 향상과 처리 속도 3배 개선**을 달성한 실제 경험을 공유합니다. 비정형 계층 구조를 가진 분류 문제를 해결하는 실용적인 접근법을 제시하고자 합니다.

### 실제 적용 환경
- **도메인**: NOL 인터파크 이커머스 플랫폼
- **데이터**: 7,170개 키워드, 비정형 계층 구조 (5-level 카테고리)
- **임베딩 모델**: Korean SBERT (snunlp/KR-SBERT-V40K-klueNLI-augSTS)
- **처리 방식**: 배치 처리 기반 벡터화 연산
- **성능 측정**: 기존 LLM + Prompting vs 새로운 Semantic Mapper

---

## 📋 본문

### 기존 해결 방안들의 한계 분석

#### 1. LLM + 프롬프트 엔지니어링 접근법

**기존 방법의 문제점:**
| 카테고리 | 일치율 | 문제점 |
| --- | --- | --- |
| vertical | 93.34% | 양호한 성능 |
| category_1 | 85.36% | 허용 가능한 수준 |
| category_2 | 63.81% | **성능 저하 시작** |
| category_3 | 55.37% | **심각한 성능 저하** |
| category_4 | 28.08% | **사용 불가 수준** |
| category_5 | 71.94% | 불안정한 성능 |

**운영상 한계:**
- **응답 시간 불안정성**: LLM 호출 시간 예측 불가능, 재시도 로직 복잡
- **유지보수 복잡성**: 프롬프트 관리에 과도한 전문 인력 필요
- **확장성 부족**: 새로운 요구사항 반영이 시간 소모적
- **비용 예측 어려움**: API 호출량에 따른 가변 비용 구조

#### 2. 전통적 머신러닝 접근법

**기존 방법:**
```python
# 문제: 계층적 구조를 평면적으로 처리
from sklearn.ensemble import RandomForestClassifier
from sklearn.feature_extraction.text import TF_IDF_Vectorizer

# TF-IDF 기반 피처 추출
vectorizer = TfIdfVectorizer(max_features=1000)
X = vectorizer.fit_transform(keywords)

# 각 카테고리별 독립적 분류
clf = RandomForestClassifier()
clf.fit(X, category_labels)
```

**핵심 한계:**
1. **알고리즘적 한계**: 텍스트의 의미적 유사성을 직접 학습하기 어려움
2. **데이터적 한계**: 카테고리별 데이터 불균형, 새로운 키워드 일반화 부족
3. **리소스적 한계**: 피처 엔지니어링 및 지속적 재학습 체계 구축 부담

### 시도한 방법: 확률 기반 트리 드리프팅 (Probabilistic Tree Drifting)

#### 1. 핵심 아이디어: 지역 최적해 vs 전역 최적해

**문제 정의:**
```
기존 Greedy 방식의 한계
Level 1에서 최고 점수 노드 선택 → Level 2에서도 이어서 선택
→ 지역 최적해에 갇힘

해결책: Top-K 후보 유지 + DFS 전역 탐색
Level 1에서 상위 K개 보존 → 모든 경로 조합 탐색
→ 전역 최적해 추구
```

**개선 방법:**
```python
# 핵심: 평균 유사도 기반 공정한 경로 평가
def select_optimal_path(valid_paths):
    """
    트리 깊이에 무관한 공정한 경로 선택
    """
    best_path = None
    best_score = -1
    
    for path in valid_paths:
        # 평균 유사도로 공정한 비교
        avg_similarity = sum(path.similarities) / len(path.similarities)
        if avg_similarity > best_score:
            best_score = avg_similarity
            best_path = path
    
    return best_path
```

#### 2. 수학적 정의 및 알고리즘

**1) 기본 표기법**
- 키워드: $k \in K$, 카테고리 계층: $L_0, L_1, ..., L_d$
- 경로: $p = (c_0, c_1, ..., c_d)$, 임베딩 함수: $\phi: K \rightarrow \mathbb{R}^n$

**2) 카테고리별 대표 벡터 계산**
$$\vec{v}_{i,j} = \frac{1}{|S_{i,j}|} \sum_{k \in S_{i,j}} \phi(k)$$

**3) 유사도 계산**
$$sim(k_{new}, c_{i,j}) = \frac{\phi(k_{new}) \cdot \vec{v}_{i,j}}{||\phi(k_{new})|| \cdot ||\vec{v}_{i,j}||} = \cos(\theta)$$

**4) 최종 경로 선택**
$$p^* = \arg\max_{p \in P_{valid}} \frac{1}{|p|} \sum_{c_i \in p} sim(k_{new}, c_i)$$

#### 3. 배치 처리 최적화 구현

**기존 방법의 비효율성:**
```python
# 문제: 키워드마다 개별 임베딩 호출
for keyword in keywords:
    embedding = model.encode(keyword)  # 개별 호출
    result = classify_single(embedding)
```

**개선 방법:**
```python
# 해결: 벡터화된 배치 처리
def predict_batch(self, keywords: List[str], K: int = 10):
    # 1. 모든 키워드를 한번에 임베딩 (병목 해결)
    all_embeddings = self.model.encode(keywords, batch_size=64)
    
    # 2. 벡터화된 코사인 유사도 계산
    for keyword, new_emb in zip(keywords, all_embeddings):
        # 매트릭스 연산으로 빠른 유사도 계산
        emb_matrix = np.stack([label_embeddings[label] for label in labels])
        similarities = util.cos_sim(new_emb, emb_matrix)[0].cpu().numpy()
        
        # 3. Top-K 필터링 + DFS 경로 탐색
        depth_topk = self._get_topk_candidates(similarities, K)
        optimal_path = self._find_optimal_path(depth_topk)
```

### 성능 측정 결과

#### 분류 정확도 비교

| 카테고리 | 기존 방법 | 개선 방법 | 성능 향상 | 개선 수준 |
|----------|-----------|-----------|-----------|-----------|
| **vertical** | 93.34% | 89.77% | -3.57% | 미세 감소 |
| **category_1** | 85.36% | 85.07% | -0.29% | 유지 |
| **category_2** | 63.81% | 81.61% | **+17.80%** | **대폭 개선** |
| **category_3** | 55.37% | 78.84% | **+23.47%** | **대폭 개선** |
| **category_4** | 28.08% | 86.88% | **+58.80%** | **혁신적 개선** |
| **category_5** | 71.94% | 85.76% | **+13.82%** | **대폭 개선** |
| **전체 평균** | **66.31%** | **84.65%** | **+18.34%** | **전반적 향상** |

#### 핵심 성과 지표
- **category_4 혁신적 개선**: 사용 불가 수준(28.08%) → 실용 가능 수준(86.88%)
- **전체 평균 18.34% 향상**: 66.31% → 84.65%로 큰 폭 개선
- **깊은 카테고리 특화**: 계층이 깊을수록 더 큰 성능 향상

#### 처리 효율성 비교

**성능 측정 결과:**

| 항목 | 기존 방법 | 개선 방법 | 성능 향상 |
|------|-----------|-----------|-----------|
| 키워드 수 | 7,170개 | 7,170개 | - |
| 소요 시간 | 512초 (예상) | 175초 (실측) | **-65.8% 단축** |
| 처리 속도 | 14.0개/초 | 40.97개/초 | **+193% 향상** |
| 배속 개선 | 1배 | **2.9배** | **거의 3배 빠름** |

**효율성 분석:**
- **배치 처리 최적화**: 개별 호출 → 벡터화된 일괄 처리
- **예측 가능한 성능**: LLM API 불안정성 제거
- **비용 효율성**: 가변 API 비용 → 고정 계산 비용

### 구현 과정의 핵심 인사이트

#### 1. 평균 유사도 선택의 논리적 근거
**문제 인식:**
```
초기 검토: 유사도 총합(sum) 방식
Path A: [0.8, 0.7, 0.6] → Sum: 2.1  (깊은 트리가 유리)
Path B: [0.9, 0.8]      → Sum: 1.7

→ 트리 깊이 편향 발생
```

**해결책 도출:**
```
개선: 평균 유사도 방식
Path A: [0.8, 0.7, 0.6] → Average: 0.7
Path B: [0.9, 0.8]      → Average: 0.85  (품질 우선 선택)

→ 구조적 편향 제거
```

#### 2. 구현상 특별한 어려움 없음
- **기술적 구현**: 기존 임베딩 모델과 벡터 연산 활용으로 안정적 구현
- **성능 최적화**: 배치 처리와 벡터화로 자연스러운 효율성 확보
- **핵심 가치**: 비용과 리소스 측면에서 큰 이득 달성

---

## 🎯 정리

### 핵심 배운 점

#### 1. 문제 해결의 다양성과 효율성 추구
- **정해진 답은 없다**: 문제를 푸는데는 여러 해결책이 존재
- **효율적 답 탐구**: 가장 효율적인 해결방안을 지속적으로 고민하는 자세
- **근본적 사고 전환**: 기존 방식의 한계를 인정하고 새로운 접근법 모색

#### 2. 수학적 근거의 중요성
- **직관을 수학으로**: 아이디어를 수학적으로 정의하여 재현 가능한 솔루션 개발
- **검증 가능성**: 수학적 근거로 성능 예측과 결과 분석 가능
- **확장성 확보**: 체계적 접근으로 다른 도메인 적용 용이

#### 3. 하이브리드 접근법의 효과
- **LLM의 한계**: 운영 복잡성과 비용 예측의 어려움
- **전통 ML의 한계**: 계층적 구조 표현의 제약
- **새로운 조합**: 임베딩 + 트리 탐색으로 두 방식의 장점 결합

### 다른 상황 적용 팁

#### 1. 계층적 분류 문제 해결 전략
```python
# 트리 구조 분류 문제 적용 가이드
1. DFS 기반 탐색 → 전역 최적해 추구
2. Top-K 필터링 → 탐색 공간 최적화
3. 평균 기반 평가 → 구조적 편향 제거
```

#### 2. 대규모 텍스트 처리 최적화
- **배치 처리**: 개별 호출 대신 벡터화된 일괄 처리
- **임베딩 캐싱**: 중복 계산 제거로 성능 향상
- **메모리 효율성**: 스트리밍 처리로 대용량 데이터 대응

#### 3. 실용성 중심 설계 원칙
- **완벽함보다 실용성**: 운영 가능한 수준에서 최적화
- **도메인 특화**: 범용보다는 특정 영역에 최적화된 접근
- **확장성 고려**: 미래 요구사항 변화에 대비한 유연한 구조

### 비즈니스 임팩트

#### 정량적 효과
- **분류 정확도**: 평균 18.34% 향상 (66.31% → 84.65%)
- **처리 효율성**: 3배 빠른 속도 (14개/초 → 40.97개/초)
- **운영 비용**: API 호출 비용 대신 일회성 계산 비용

#### 정성적 효과
- **운영 안정성**: 예측 가능한 처리 시간으로 시스템 안정성 확보
- **확장성**: 새로운 키워드 추가 시 즉시 처리 가능
- **유지보수성**: 수학적 근거 기반으로 명확한 디버깅 가능

---

## 📚 참고자료

### 기술 문서
- [SentenceTransformers Documentation](https://www.sbert.net/)
- [Korean SBERT Model Hub](https://huggingface.co/snunlp/KR-SBERT-V40K-klueNLI-augSTS)
- [Polars DataFrame Performance Guide](https://pola-rs.github.io/polars/)

### 알고리즘 및 이론
- **Tree Search Algorithms**: Depth-First Search for hierarchical classification
- **Embedding Similarity**: Cosine similarity in high-dimensional spaces
- **Batch Processing**: Vectorized operations for performance optimization

---

*실험 일시: 2025-08-12*  
*환경: NOL 인터파크, Korean SBERT, 7,170개 키워드*