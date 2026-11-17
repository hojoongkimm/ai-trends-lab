# 🤖 PaperOrchestra: Manuscript Assembly as a Five-Stage Agent Pipeline — Reverse-Engineered Benchmark, Search-Grounded Citation Registry, and Hill-Climbing Refinement

- **Repository**: [google-research/paper-orchestra](https://github.com/google-research/paper-orchestra)
- **Stars**: ⭐ 1,543 (forks 239, created 2026-05, last commit 2026-11-15)
- **Category**: `Agent` / Multi-Agent Scientific Writing Pipeline
- **Organization**: Google Cloud AI Research (not an officially supported Google product)
- **License / Runtime**: Apache-2.0 / Python 3.11, Gemini·OpenAI·Vertex backends, LaTeX toolchain (`pdflatex`) and a Streamlit frontend
- **Key Themes**: five specialized agents with a fixed dataflow, temporal-cutoff literature retrieval validated against Semantic Scholar, BibTeX and citation-map generation, a reviewer-in-the-loop refinement loop with hill-climbing acceptance, autorater suite (citation F1, literature-review rubric, dual peer-review simulators, order-swapped side-by-side)

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

자율 연구 에이전트는 대개 실험 루프와 집필 모듈이 한 몸으로 붙어 있다. 그래서 사람이 이미 가진 재료(아이디어 메모, 실험 로그)를 넣어 논문만 뽑는 일은 오히려 어렵다. 이 저장소가 겨냥하는 병목은 네 가지다.

1. **비정형 입력의 구조화(Unstructured-to-Structured Gap)**: 입력이 `idea.md`와 `experimental_log.md` 두 개의 자유 서술이라는 점이다. 실험 로그에는 수치가 표로 있지만 서술 산문은 없고, 아이디어 문서에는 동기가 있지만 결과는 없다. 두 문서에서 절별 계획·그림 계획·검색 계획을 동시에 뽑아내야 한다.
2. **인용의 사실성(Citation Grounding)**: 파라메트릭 기억으로 참고문헌을 쓰면 존재하지 않는 논문이 생긴다. 반대로 검색 결과를 그대로 붙이면 시점이 섞인다. 검증된 서지 레코드와 시간 상한을 동시에 강제해야 한다.
3. **부분 실패의 전파(Compile Cascade)**: LaTeX는 조판 가능성이라는 이진 제약을 갖는다. 앞 단계의 작은 오류가 PDF 컴파일 실패로 증폭되면 그 뒤 평가는 전부 무의미해진다.
4. **평가 기준의 부재(Evaluation Vacuum)**: "좋은 논문"의 기준이 없으면 개선 루프도 성립하지 않는다. 이 저장소는 벤치마크(PaperWritingBench)와 autorater를 파이프라인과 함께 배포해 이 문제를 먼저 해결한다.

이 병목들에 대한 응답은 **역할 분해 + 검증된 검색 + 컴파일 가능성 제약 + 점수 기반 개선**이다. 각 요소는 개별적으로 새롭지 않지만, 네 가지를 한 파이프라인에 묶고 사람 평가까지 붙여 정량화한 것이 이 저장소의 실질적 기여다.

---

## 2. 아키텍처 및 수학적 정식화 (Architecture & Formulation)

### A. 과업의 형식화

파이프라인은 다섯 입력을 하나의 산출물로 사상하는 함수 $W$다. 아이디어 요약 $\mathcal{I}$, 실험 로그 $\mathcal{E}$, LaTeX 템플릿 $\mathcal{T}$, 학회 지침 $\mathcal{G}$, 선택적 그림 집합 $\mathcal{F}$에 대해

$$
W : (\mathcal{I}, \mathcal{E}, \mathcal{T}, \mathcal{G}, \mathcal{F}) \mapsto P = (P_{\text{tex}}, P_{\text{pdf}})
$$

$\mathcal{F} = \varnothing$이면 그림까지 스스로 생성한다는 조건이 붙는다. 즉 이 함수는 총함수가 아니라, 조판 실패 시 정의되지 않는 부분함수다. 그래서 구현은 전체를 3회 재시도 루프로 감싼다(`max_n_tries = 3`). 조판 가능성은 확률적 제약이고, 재시도는 그 제약을 만족하는 표본을 다시 뽑는 절차로 읽을 수 있다.

### B. 다섯 단계의 데이터흐름

1단계 `OutlineAgent`는 절별 계획, 시각화 계획, 문헌 검색 전략을 담은 JSON 개요를 만든다. 2단계 `PlottingAgent`와 3단계 `HybridLiteratureAgent`는 병렬로 돌고, 4단계 `SectionWritingAgent`가 개요·인용 지도·그림 정보를 받아 원고를 채우며, 5단계 `ContentRefinementAgent`가 PDF를 컴파일해 리뷰 점수를 올린다. 단계 간 인계물은 전부 파일이다(개요 JSON, `.bib`, `citation_map.json`, `figures/info.json`). 컨텍스트가 아니라 디스크가 상태를 소유하므로, 한 단계가 실패하면 그 지점부터 다시 실행할 수 있다.

### C. 검색-검증 2단 구조와 시간 상한

문헌 단계는 검색을 두 성질로 분리한다. **후보 발견**은 검색 그라운딩 LLM 호출 10개를 스레드 풀로 동시에 돌려 후보 풀을 만든다. **존재 검증**은 후보를 하나씩 Semantic Scholar 제목 검색으로 조회하고 **호출 사이에 1.0초를 강제**한다. 처리량은 정확도에 종속되므로 병렬화할 수 없고, 그래서 이 단계만 순차다. 레지스트리는 정규화 제목으로 중복을 제거한다.

시간 상한은 누출 방지 장치다. 기준 시점 $\tau$에 대해 허용 집합은

$$
\mathcal{R}(\tau) = \lbrace p : \mathrm{date}(p) \le \tau \rbrace, \qquad \mathrm{date}(p) \ne \varnothing
$$

이고, 게시일이 없거나 $\tau$를 넘는 후보는 조용히 버려진다. CLI 기본값이 `2024-11`이라는 점이 중요하다. 최신 모델로 2025년 논문을 쓰면 학습 데이터 누출이 의심되므로, 기준 시점을 명시적으로 과거에 두어 "그때 알 수 있었던 문헌"만 쓰게 만든다. 초록은 1,500자에서 절단해 컨텍스트를 균등화한다.

### D. 인용의 정밀도-재현율 교환

생성 참고문헌 집합 $\hat{R}$과 정답 집합 $R$에 대해

$$
P = \frac{\lvert \hat{R} \cap R \rvert}{\lvert \hat{R} \rvert}, \quad
R_{\text{rec}} = \frac{\lvert \hat{R} \cap R \rvert}{\lvert R \rvert}, \quad
F_1 = \frac{2PR_{\text{rec}}}{P + R_{\text{rec}}}
$$

정답 인용은 P0(반드시 인용: 직접 비교한 베이스라인, 사용 데이터셋, 의존 방법)과 P1(배경·보조)으로 분할되고, P0 재현율이 실질 지표다. 이 분할이 없으면 $F_1$은 **인용을 적게 쓸수록 유리해지는** 지표가 된다. 실제로 단일 호출 베이스라인은 인용 11.46편, $F_1$ 28.03인데 P1 재현율은 2.77에 그친다. 반면 이 시스템은 47.98편, $F_1$ 29.65, P1 재현율 15.85다. $F_1$만 보면 차이가 1.6포인트지만, 커버리지는 4배 이상 차이 난다. 지표 설계가 결론을 좌우하는 전형이다.

### E. 개선 루프의 수용 규칙과 종료성

5단계는 자기 논문을 리뷰어 모델에 넣어 9개 축(Originality, Quality, Clarity, Significance, Soundness, Presentation, Contribution, Overall, Confidence) 점수를 받고, 점수가 오르는 수정만 남긴다. 수용 조건은

$$
\text{accept}(s_t, s_{t+1}) =
\begin{cases}
\text{true} & S_{t+1} \gt S_t \\
\text{true} & S_{t+1} = S_t \ \text{and} \ \sum_{a \notin \lbrace \text{Overall}, \text{Confidence} \rbrace} \Delta_a \ge 0 \\
\text{false} & \text{otherwise}
\end{cases}
$$

이고, 거절이면 직전 LaTeX로 되돌리고 루프를 즉시 멈춘다. 반복 상한은 3회다. 따라서 점수열은 단조 비감소이고 반복 수는 유한하므로 종료성은 보장된다. 문제는 목적함수가 고정되어 있지 않다는 점이다. 같은 LaTeX를 다시 채점해도 판정자 표본 추출 때문에 점수가 흔들리므로, "상승"은 참 상승이 아니라 잡음 있는 추정치의 상승이다. 저자들이 리뷰 축에서 Overall과 Confidence를 비교에서 제외한 것은 이 잡음을 줄이려는 장치로 읽힌다.

수용된 수정의 효과는 분리 측정되어 있다. 정제 후 원고는 정제 전 원고를 상대로 79–81% 승률, 패배 0%를 기록하고, 시뮬레이션 채택률은 CVPR +19포인트, ICLR +22포인트, 종합 점수는 +0.88과 +1.61 올랐다. 정제가 조판·명료성·발표 축을 건드리기 때문에 생기는 폭이다.

### F. 컴파일을 제약으로 쓰는 탐색

정제 루프는 수정본을 컴파일하고, PDF가 없으면 그 후보를 버리고 이전 버전을 유지한다. 즉 탐색 공간은 **컴파일 성공한 LaTeX 프로그램**으로 제한된다. 이 제약은 점수보다 우선하며, 덕분에 최종 산출물은 항상 열리는 PDF다. 실패 모드도 단순해진다. 남는 위험은 "조판은 되지만 내용이 틀린" 문서이고, 그것은 점수 축이 아니라 사실 검증 축의 문제다.

페이지 이미지를 격자로 잘라 리뷰어에 함께 넣는 설계도 이 계열이다. 레이아웃 결함(넘친 표, 고아 캡션)은 텍스트만으로는 보이지 않는다.

### G. 정량 결과와 그 해석

PaperWritingBench는 CVPR 2025와 ICLR 2025 논문 각 100편에서 재료를 역설계해 만들었다. PDF는 MinerU와 PDFFigures로 파싱하고, 아이디어 요약과 실험 로그는 LLM으로 역생성하되 인용·URL·그림 참조를 모두 지워 자기완결적으로 만든다. 아이디어는 Sparse(고수준 요약)와 Dense(형식 정의와 수식 포함) 두 난이도로 제공된다.

| 지표 | 단일 호출 | AI Scientist-v2 | PaperOrchestra | 사람 정답 |
| :--- | ---: | ---: | ---: | ---: |
| 평균 인용 수 (CVPR / ICLR) | 11.46 / 9.75 | 14.18 / 13.71 | 47.98 / 45.73 | 58.52 / 59.18 |
| 인용 $F_1$ (CVPR / ICLR) | 28.03 / 23.16 | 17.15 / 15.80 | 29.65 / 27.76 | — |
| P0 재현율 (CVPR / ICLR) | 60.84 / 48.41 | 36.12 / 33.66 | 63.58 / 54.48 | — |
| P1 재현율 (CVPR / ICLR) | 2.77 / 3.09 | 3.26 / 3.07 | 15.85 / 16.43 | — |
| 문헌 리뷰 종합 (CVPR, Gemini 판정) | 43.04 | 45.43 | 78.30 | 71.08 |
| ScholarPeer 채택률 (CVPR / ICLR) | 71 / 72 | 70 / 64 | 84 / 81 | 86 / 94 |
| LLM 호출 수 · 평균 지연 | 1 · 3.3분 | 40–45 · 35.1분 | 60–70 · 39.6분 | — |

사람 평가는 연구자 11명이 40편을 4개 조건과 비교한 180쌍이다. 문헌 리뷰 절대 승률 마진 50–68%, 논문 전체 품질 14–38%다. 전체 품질에 대한 사람-AI 판정 상관은 Pearson $r = 0.6458$(GPT-5)와 $0.5672$(Gemini)로 쓸 만하지만, **문헌 리뷰는 $r = 0.0868$(Gemini)과 $0.2764$(GPT-5)로 사실상 무상관**이다. 저자들의 해석은 LLM 판정자가 "문제-격차-해결" 같은 형식 신호를 보상하는 반면 사람은 서사적 사실성에 가중치를 둔다는 것이다. 즉 이 저장소가 자랑하는 문헌 리뷰 승률은 사람 기준으로는 측정되지 않은 부분이 크다.

---

## 3. 구조적 비교 (Comparative Positioning)

| 항목 | PaperRobot (2019) | AI Scientist-v2 | CycleResearcher | data-to-paper | PaperOrchestra |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 독립 집필 도구 | ✗ | ✗ | ✗ | ✓ | ✓ |
| 비정형 입력 허용 | ✗ | ✓ | ✗ (BibTeX 요구) | 부분 | ✓ |
| 겨냥한 문헌 검색 | ✗ | ✓ | ✗ | ✗ | ✓ |
| 개념 도해 생성 | ✗ | ✗ | ✗ | ✗ | ✓ |
| 인용 사실 검증 | ✗ | 부분 | ✗ | ✗ | Semantic Scholar 조회 |
| 개선 루프 | ✗ | 3회 성찰 | RL 보상 | ✗ | 점수 기반 3회 + 롤백 |
| 벤치마크 공개 | ✗ | ✗ | ✗ | ✗ | 200편 역설계 |

`agent/` 카테고리 안에서 위치는 "오케스트레이션 런타임(ruflo, CowAgent) → SOP 실행(LoopForge) → **산출물 생성 파이프라인(PaperOrchestra)**"의 마지막 칸이다. 앞의 둘이 실행을 조율하는 데 집중한다면, 이쪽은 한 편의 완성품을 만들고 평가까지 붙인다. 강점은 검증된 서지와 조판 제약이고, 약점은 평가가 전부 LLM 판정자에 기대며 그 판정자와 사람 사이의 상관이 축마다 크게 다르다는 점이다.

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **자기 판정 루프의 순환성**: 개선 신호가 같은 계열 모델의 리뷰 점수다. 점수가 올라도 리뷰어가 좋아하는 형식이 강화될 뿐일 수 있다. 수용 규칙이 Overall과 Confidence를 배제해 잡음을 줄이지만, 편향은 줄이지 못한다.
2. **문헌 리뷰 autorater의 낮은 사람 상관**: Gemini 판정 기준 $r = 0.0868$은 사실상 무정보에 가깝다. 자동 지표로 보고된 78.30점이 사람의 71.08점과 같은 것을 재고 있다고 보기는 어렵다.
3. **$F_1$의 규모 의존성**: 인용 수가 5배 다르면 $F_1$ 비교는 성립하지 않는다. 저자들도 지적하지만, 논문 표에서 $F_1$이 같은 열에 놓이면 독자는 그대로 비교하게 된다. P0/P1 재현율을 주지표로 두는 편이 정직하다.
4. **부품 의존**: 그림 생성은 PaperBanana, 리뷰는 AgentReview, 판정은 ScholarPeer라는 외부 프레임워크에 위임한다. 그림 환각에 대한 직접 통제가 없다고 부록에서 인정한다. 부품이 바뀌면 보고된 수치의 의미도 바뀐다.
5. **데이터셋 미공개**: 벤치마크는 저장소에 없고 "추후 공개" 상태다. 재현성은 논문 수준에서만 성립한다.
6. **모델 고정 비용**: 1편당 60–70회 LLM 호출과 39.6분. 개선 루프 3회가 호출의 상당 부분을 차지하므로, 루프를 늘리면 비용이 선형으로 늘고 이득은 체감한다.
7. **사실 검증의 부재**: 실험 로그의 수치가 원고 표에 옮겨지는지는 형식적으로만 보장된다. 로그에 없는 수치를 서술로 채우는 실패는 점수 축에서 벌점이 없다.
8. **재시도 은폐**: 3회 재시도 루프는 예외를 삼키고 다시 시작한다. 실패율 자체가 보고되지 않으므로, 성공한 실행만 통계에 남는 생존 편향이 있을 수 있다.
9. **벤치마크 누출 위험**: 재료를 익명화하고 인용을 제거했지만, 원 논문이 학습 데이터에 있었을 가능성은 남는다. 저자들도 미공개 연구로 벤치마크를 다시 만들어야 한다고 적는다.
10. **언어·학회 편향**: 템플릿이 CVPR 2025와 ICLR 2025 두 개다. 문자열 기반 지침 준수(페이지 수, 절 구성)는 잘 되지만, 다른 학회의 관행은 반영되지 않는다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **컴파일을 제약으로 쓰는 설계**: 생성 결과를 하나의 검증기(LaTeX 컴파일)로 걸러내고, 실패 시 이전 상태로 롤백하는 구조는 코드 생성·스키마 생성 등 검증기가 존재하는 모든 생성 파이프라인에 그대로 이식된다. 제약 만족률을 측정하면 파이프라인 건강도를 단일 숫자로 감시할 수 있다.
- **시간 상한 $\tau$의 명시**: 벤치마크에서 지식 누출을 막는 가장 값싼 장치는 모델 교체가 아니라 기준 시점의 고정이다. 평가 설계에서 $\tau$를 하이퍼파라미터로 취급하고 보고하는 관행이 필요하다.
- **검색과 검증의 분리**: 넓게 병렬로 발견하고, 좁게 순차로 확인한다. 이 분리는 속도 제약(레이트 리밋)이 있는 모든 지식 파이프라인의 기본형이다. 병렬 폭 10과 검증 간격 1초라는 구체값은 재현 가능한 출발점이다.
- **P0/P1 분할 지표**: 인용 평가에서 "필수"와 "보조"를 나누지 않으면 지표가 절약을 보상한다. 이 분할은 요약·근거 인용 평가 전반에 적용 가능한 일반 원칙이다.
- **판정자 검증의 필요성**: 축별 사람 상관을 보고하지 않는 LLM 판정 결과는 신뢰 구간이 없다. 이 저장소는 축별 상관을 공개해 스스로의 약점을 드러냈고, 그 점이 오히려 방법론적 자산이다.
- **후속 연구 방향**: (i) 리뷰 점수 대신 검증 가능한 사실 주장(수치 일치율)을 목적함수로 쓰는 개선 루프, (ii) 조판 실패율을 1급 지표로 보고하기, (iii) 문헌 리뷰 평가에서 사람과 LLM의 불일치 원인을 형식 신호와 사실성으로 분해하기, (iv) 재료 밀도(Sparse/Dense)와 산출 품질의 상호작용을 회귀적으로 모델링하기, (v) 미공개 연구 재료로 누출 없는 벤치마크 구성하기.
