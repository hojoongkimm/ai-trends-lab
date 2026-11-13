# ⚡ LM-Evaluation-Harness: Rigorous Few-Shot Language Model Evaluation Framework

- **Repository**: [EleutherAI/lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness)
- **Stars**: ⭐ 14,500+
- **Category**: `Harness` / `LLM Evaluation & Benchmarking Systems`
- **Conference / Venue**: EleutherAI Technical Framework / Hugging Face Open LLM Leaderboard Core Engine
- **Organization**: EleutherAI
- **Key Themes**: Log-Likelihood Evaluation, Generative Evaluation, Pass@k, Contamination Auditing, Standardized Prompts

---

## 1. 연구 배경 및 평가 표준의 부재 (Research Problem)

초기 거대 언어 모델(LLM) 연구들은 각 연구팀마다 서로 다른 프롬프트 형식, 공백/구두점 처리 방식, 토크나이저 확률 정규화 기법을 사용하여 **모델 간 비교 결과의 재현성(Reproducibility)이 심각하게 훼손**되었습니다:
1. **프롬프트 민감도(Prompt Brittleness)**: 공백 한 칸, 지시문 미세 수정만으로도 MMLU 정확도가 $5\sim 10\%$ 변동함.
2. **로그 확률 vs 생성 평가의 불일치**: 객관식 문제에서 Log-Likelihood 기반 선택과 Greedy Generation 기반 선택 간의 점수 왜곡.

`lm-evaluation-harness`는 **수학적 평가 지표(Log-Likelihood, Length-Normalized Perplexity, Pass@k)와 프롬프트 템플릿을 코드 레벨에서 완전히 표준화**하여, 전 세계 학계(NeurIPS, ICML, ACL)와 리더보드의 공인 벤치마크 표준으로 확립되었습니다.

---

## 2. 수학적 평가 원리 (Mathematical Formulation of Metrics)

### A. Likelihood-based Multi-Choice Scoring
프롬프트 $X$와 후보 정답 $Y = (y_1, \dots, y_m)$에 대해, 모델의 조건부 로그 우도(Conditional Log-Likelihood)를 계산합니다:

$$\log P(Y \mid X) = \sum_{t=1}^m \log P(y_t \mid X, y_{<t})$$

정답 선택지들의 토큰 길이 편향을 제거하기 위해 **길이 정규화(Length Normalization)**를 적용합니다:

$$\text{Score}_{\text{norm}}(Y \mid X) = \frac{1}{m^\alpha} \sum_{t=1}^m \log P(y_t \mid X, y_{<t})$$

### B. Unbiased Code Generation Metric (Pass@k)
코딩 벤치마크(HumanEval, MBPP)에서 $n$개 샘플 중 $c$개가 정답 테스트를 통과했을 때, 편향 없는(Unbiased) Pass@k 기댓값을 콤비네이션 수식으로 계산합니다 (Chen et al., 2021):

$$\text{Pass}@k = \mathbb{E} \left[ 1 - \frac{\binom{n - c}{k}}{\binom{n}{k}} \right]$$

이 수식은 나이브한 추정치의 고분산(High Variance) 문제를 수학적으로 상쇄합니다.

---

## 3. 대표 벤치마크 스위트 구성 (Benchmark Taxonomy)

| 벤치마크 태스크 | 평가 대상 역량 | 평가 메커니즘 | 지표 |
| :--- | :--- | :---: | :---: |
| **MMLU** (57개 세부 분야) | 학술 지식 및 종합 인지 추론 | 5-shot Log-Likelihood | Accuracy (Acc_norm) |
| **GSM8k** | 초등 다단계 수학적 사고(CoT) | 8-shot Strict Generation | Exact Match (EM) |
| **HumanEval / MBPP** | 파이썬 코드 합성 및 실행 정확성 | 0-shot Sandbox Execution | Pass@1, Pass@10 |
| **ARC (Challenge)** | 초등/중등 고급 과학 상식 추론 | 25-shot Log-Likelihood | Acc_norm |
| **TruthfulQA** | 할루시네이션 및 편향/오개념 저항력 | 0-shot MC2 Probability | MC2 Score |

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **데이터 오염(Data Contamination)**: 최신 파운데이션 모델들의 대규모 사전 훈련 데이터에 MMLU/GSM8k의 테스트 데이터가 유출되어 점수가 과대평가되는 오염 문제를 자체 감지하기 어려움 (별도 n-gram 오염 검사 필요).
2. **CoT 생성 평가의 채점 취약성**: 복잡한 정답 문자열 파싱(Regex Parsing) 실패 시 실제 논리적 정답을 맞혔음에도 오답 처리되는 파싱 노이즈 존재.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **파인튜닝 및 양자화 모델의 성능 검증**: 자체 학습한 도메인 어댑테이션 모델이나 INT4/FP8 양자화 모델의 성능 저하를 학회 제출용 공인 지표로 객관화.
- **vLLM 백엔드 가속 연동**: 수천 개 문항 평가 시 vLLM 엔진을 직접 바인딩하여 수 분 내에 전체 벤치마크 측정 완료.

