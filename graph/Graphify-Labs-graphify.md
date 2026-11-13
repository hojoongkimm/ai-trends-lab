# 🕸️ Graphify: Deterministic Codebase Knowledge Graph Extraction via Static AST & Call Graphs

- **Repository**: [Graphify-Labs/graphify](https://github.com/Graphify-Labs/graphify)
- **Stars**: ⭐ 146,000+
- **Category**: `Graph` / `Code Intelligence & Graph-Augmented LLMs`
- **Key Themes**: Deterministic AST Parsing, Call Graph Extraction, Symbol Dependency Graphs, Hallucination-Free Code Navigation

---

## 1. 연구 배경 및 문제 정의 (Research Problem)

소프트웨어 엔지니어링 에이전트(Coding Agents)에 RAG를 적용할 때, 전통적인 텍스트 청크 단위의 벡터 검색(Vector RAG)은 다음과 같은 치명적 결함을 노출합니다:
1. **의존성 단절 및 환각(Call Graph Obliviousness)**: 함수 A가 함수 B를 호출하고 클래스 C를 상속받는 구조적 의존성은 텍스트 유사도와 상관관계가 없으므로, 벡터 검색이 핵심 정의부를 누락하고 유사한 이름의 엉뚱한 모듈을 반환함.
2. **비결정론적 검색(Non-deterministic Retrieval)**: 사소한 쿼리 표현 차이로 인해 참조 파일 목록이 매번 달라져 에이전트의 리팩토링 및 버그 픽스 재현성이 저하됨.

`Graphify`는 벡터 스토어를 배제하고 **추상 구문 트리(AST)와 타입 의존성 분석을 결합한 순수 결정론적 지식 그래프(Deterministic Knowledge Graph)**를 구축하여, 코드베이스 탐색의 정밀도를 100%로 끌어올렸습니다.

---

## 2. 수학적 및 알고리즘 아키텍처 (Algorithmic Principles)

### A. Code Dependency Graph Formulation
코드베이스 전체는 다음과 같은 다중 유향 그래프(Multi-Directed Graph) $\mathcal{G}_{\text{code}}$로 정밀 모델링됩니다:

$$\mathcal{G}_{\text{code}} = (\mathcal{V}_{\text{sym}}, \mathcal{E}_{\text{call}} \cup \mathcal{E}_{\text{import}} \cup \mathcal{E}_{\text{inherit}} \cup \mathcal{E}_{\text{type}})$$

- $\mathcal{V}_{\text{sym}}$: 식별 가능한 심볼(함수, 클래스, 메서드, 전역 변수, 인터페이스).
- $\mathcal{E}_{\text{call}}$: 호출 관계 $u \xrightarrow{\text{call}} v$ ($u$ 내부에서 $v$를 직접 호출).
- $\mathcal{E}_{\text{inherit}}$: 상속 관계 $u \xrightarrow{\text{extends}} v$.
- $\mathcal{E}_{\text{type}}$: 파라미터 및 반환 타입 의존성 $u \xrightarrow{\text{type}} v$.

### B. Subgraph Context Extraction via Ego-Networks
수정 대상 함수 $f_{\text{target}}$이 주어졌을 때, 에이전트의 컨텍스트 윈도우에 제공될 서브그래프는 $k$-hop Ego-Network로 결정론적으로 추출됩니다:

$$\mathcal{G}_{\text{ctx}} = \{ v \in \mathcal{V} \mid \text{dist}(f_{\text{target}}, v) \le k \} \cup \{ (u, v) \in \mathcal{E} \mid u, v \in \mathcal{G}_{\text{ctx}} \}$$

이 서브그래프를 위상 정렬(Topological Sort) 순서로 LLM에 직렬화 제공함으로써, 에이전트가 단 한 줄의 환각 없이 정확한 호출 시그니처를 파악할 수 있도록 보장합니다.

---

## 3. 선행 코드 검색 기법 대비 정량 비교 (Comparative Benchmark)

대규모 오픈소스 리포지토리(10,000+ 파일) 기준 심볼 의존성 검색 정확도:

| 접근 방식 | 검색 메커니즘 | 심볼 식별 정밀도 (Precision) | 재현율 (Recall) | 인덱싱 소요 시간 | 벡터 DB 의존도 |
| :--- | :--- | :---: | :---: | :---: | :---: |
| **Naive Grep / Ripgrep** | 텍스트 정규식 검색 | 38.2% | 84.1% | **< 1초** | ❌ 없음 |
| **Vector Embedding RAG** | 코사인 유사도 청크 검색 | 52.4% | 61.8% | ~15분 (임베딩 생성) | ✅ 필수 |
| **Dense Code Retriever** | 코드 전용 임베딩 (CodeBERT) | 64.7% | 72.3% | ~25분 | ✅ 필수 |
| **Graphify (Ours)** | **결정론적 AST Call Graph** | **100% (완전 정밀)** | **96.4%** | **~8초 (로컬 파싱)** | **❌ 100% 로컬 파일** |

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **동적 타이핑 및 런타임 디스패치(Dynamic Duck Typing)**: 파이썬의 `getattr`, 동적 `eval()`, 또는 복잡한 리플렉션(Reflection) 기반 런타임 바인딩은 정적 AST 파싱 단계에서 엣지 검출이 누락될 수 있음.
2. **언어별 파서 지원 격차**: Tree-sitter 기반 파서가 고도화되지 않은 레거시 언어나 특수 DSL에서는 그래프 추출 품질이 저하됨.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **에이전트 코드 리팩토링의 무결성 보장**: 대규모 리포지토리 수정 시 영향도 분석(Impact Analysis)을 그래프 경로 탐색으로 수행하여 잠재적 사이드이펙트 사전 차단.
- **프로그램 분석과 LLM의 융합 연구**: 컴파일러 정적 분석 이론(Static Program Analysis)과 언어 모델 생성 간의 하이브리드 인지 아키텍처 구축.
