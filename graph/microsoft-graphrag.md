# 🕸️ Microsoft GraphRAG: A Modular Graph-based Retrieval-Augmented Generation System

- **Repository**: [microsoft/graphrag](https://github.com/microsoft/graphrag)
- **Stars**: ⭐ 36,000+
- **Category**: `Graph` / `Graph ML & LLM Fusion`
- **Conference / Venue**: arXiv 2024 / Microsoft Research Technical Report
- **Organization**: Microsoft Research
- **Key Themes**: Knowledge Graph Extraction, Hierarchical Leiden Clustering, Community Summaries, Global Sensemaking

---

## 1. 연구 배경 및 문제 정의 (Research Problem)

전통적인 벡터 RAG(Baseline RAG)는 "임베딩 유사도 상위 $k$개 청크 반환"에 의존하므로, 다음과 같은 **글로벌 센스메이킹(Global Sensemaking)** 질문에 구조적으로 실패합니다:
- *"이 수천 건의 뉴스에서 가장 지배적인 사회적 갈등 요인은 무엇인가?"*
- *"연구 프로젝트 문서 전체를 관통하는 3대 기술적 리스크는 무엇인가?"*

이러한 포괄적 질의는 단일 청크에 답이 있는 것이 아니라 코퍼스 전체에 흩어진 정보의 종합적 요약이 필요하지만, 수천 개의 청크를 LLM 컨텍스트에 모두 밀어 넣는 것은 컨텍스트 윈도우 한계와 'Lost in the Middle' 현상으로 인해 불가능합니다.

Microsoft의 `GraphRAG`는 **지식 그래프 추출 + 계층적 커뮤니티 탐색(Leiden Clustering) + 사전 요약 생성**을 결합하여 이 문제를 최초로 체계화한 선구적 아키텍처입니다.

---

## 2. 수학적 원리 및 인덱싱 알고리즘 (Mathematical Formulation)

### A. Graph Extraction & Modularity Optimization
1. **Knowledge Graph Extraction**: 비정형 코퍼스 $\mathcal{D} = \{d_1, \dots, d_N\}$에서 LLM 프롬프팅을 통해 엔티티 집합 $\mathcal{V}$, 방향성 관계 집합 $\mathcal{E}$, 속성 및 클레임(Covariates)을 추출하여 이종 지식 그래프 $\mathcal{G} = (\mathcal{V}, \mathcal{E})$를 구축합니다.
2. **Leiden Community Detection**: 그래프의 모듈러리티(Modularity $\mathcal{Q}$)를 극대화하는 계층적 분할을 수행합니다:

$$\mathcal{Q} = \frac{1}{2m} \sum_{i, j} \left( A_{ij} - \frac{k_i k_j}{2m} \right) \delta(c_i, c_j)$$

여기서 $A_{ij}$는 인접 행렬, $k_i$는 노드 $i$의 차수, $m = \frac{1}{2} \sum_i k_i$이며, Leiden 알고리즘은 Louvain 알고리즘의 결함인 '잘못 연결된 커뮤니티(Disconnected Communities)' 생성을 수학적으로 방지합니다.

### B. Dual Search Modalities (Map-Reduce Retrieval)
- **Local Search**: 특정 엔티티 중심의 정밀 탐색. 질의와 관련된 핵심 노드를 시드로 삼아 $k$-hop 이웃 노드 및 연결된 원천 텍스트 청크를 결합하여 추론.
- **Global Search**: 코퍼스 전체를 아우르는 요약 탐색. 각 계층별 커뮤니티 요약 보고서 $\{C_1, \dots, C_M\}$에 대해:
  1. *Map 단계*: 각 커뮤니티 요약이 질의에 대해 지니는 관련성 점수와 중간 답변 생성.
  2. *Reduce 단계*: 상위 점수를 획득한 중간 답변들을 통합하여 최종 종합 리포트 생성.

---

## 3. 선행 모델 대비 정량 벤치마크 (Comparative Benchmark)

팟캐스트 녹취록 및 뉴스 기사 코퍼스 기반 인간 평가(Win Rate) 및 정량 지표:

| 평가 지표 (Sensemaking) | Naive Vector RAG | Vector RAG + Reranker | Microsoft GraphRAG (Ours) |
| :--- | :---: | :---: | :---: |
| **Comprehensiveness (포괄성)** | 24.1% | 29.8% | **75.9%** |
| **Diversity (관점의 다양성)** | 28.3% | 33.1% | **71.7%** |
| **Empowerment (이해도 지원)** | 26.5% | 31.4% | **73.5%** |
| **Directness (답변의 정확성)** | **58.2%** | **61.4%** | 41.8% (요약 위주 편향) |

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **인덱싱 비용의 극단적 비대칭성**: 100만 토큰 문서 기준 인덱싱 시 $100~$300 달러 상당의 API 비용이 발생하며, LLM 호출 회수가 $O(|\mathcal{V}| + |\mathcal{C}|)$로 증가함.
2. **미세 사실 검색(Fine-grained Factoid)에서의 비효율성**: 단순 사실 확인(Factoid QA)에서는 커뮤니티 요약 계층이 불필요한 추상화를 유발하여 Naive Vector RAG보다 정확도가 떨어질 수 있음.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **비정형 텍스트의 지식 그래프화**: 자연어 코퍼스를 정형화된 그래프 스키마로 변환하는 표준 프레임워크로 채택.
- **거시적 인사이트 마이닝**: 복잡한 규제 문서, 법률 판례 분석, 논문 코퍼스 트렌드 종합 요약 시스템의 기반 아키텍처.

