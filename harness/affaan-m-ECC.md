# ⚡ ECC: Systems Architecture and Optimization for Frontier Coding Agent Harnesses

- **Repository**: [affaan-m/ECC](https://github.com/affaan-m/ECC)
- **Stars**: ⭐ 300,000+
- **Category**: `Harness` / `Agent Runtime Systems & Security Guardrails`
- **Key Themes**: Agent Harness Design, Dynamic Context Budgeting, Sandboxed Tool Execution, Fault Recovery

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

프론티어 모델(Claude 3.5 Sonnet, GPT-4o, Codex)의 코드 생성 능력이 급격히 향상되었음에도 불구하고, 실제 소프트웨어 리포지토리에서 자율 코딩 에이전트를 가동할 때 다음과 같은 **런타임 시스템 레벨의 실패(Harness Failures)**가 빈번하게 발생합니다:
1. **컨텍스트 포화 및 토큰 낭비(Context Window Exhaustion)**: 빌드 로그, 에러 스택 트레이스, 대규모 소스 코드가 무차별적으로 컨텍스트에 누적되어 최대 토큰 한도에 도달하거나 핵심 목표가 밀려남.
2. **도구 선택 환각(Tool Selection Ambiguity)**: 수십 개의 도구 정의(Tool Schema)가 시스템 프롬프트에 상주할 경우, 모델의 어텐션이 분산되어 부적절한 도구를 호출하거나 인자(Argument) 형식이 깨짐.
3. **위험한 부작용 및 무한 루프**: 비가역적인 시스템 명령어(`rm -rf`, 무한 재귀 빌드) 실행 및 비정상 루프 진입.

`ECC`는 언어 모델 주변을 감싸는 **에이전트 런타임 하네스(Agent Runtime Harness)**를 시스템 공학적으로 설계하여, 신뢰성(Reliability)과 보안성(Security)을 수학적·체계적으로 보장하는 프레임워크입니다.

---

## 2. 수학적 원리 및 아키텍처 (Mathematical Formulation & System Controls)

### A. Dynamic Context Budget Allocation
전체 컨텍스트 윈도우 $C_{\max}$ 내에서 각 인지 모듈의 토큰 점유율을 동적으로 할당하는 최적화 문제입니다:

$$C_{\text{system}} + C_{\text{memory}} + C_{\text{skills}}(t) + C_{\text{history}}(t) \le C_{\max}$$

ECC는 **Dynamic Skill Registry**를 통해 현재 사용자 질의 $q$와의 임베딩 유사도 $S(\text{Skill}_k, q) \ge \tau$를 만족하는 상위 $K$개의 스킬 스키마만 온디맨드로 인젝트합니다:

$$C_{\text{skills}}(t) = \sum_{k \in \text{TopK}} \text{len}(\text{Schema}_k) \ll \sum_{\text{all}} \text{len}(\text{Schema}_k)$$

이를 통해 도구 정의에 소모되는 고정 토큰 비용을 최대 70% 절감하고 모델의 도구 선택 정확도를 극대화합니다.

### B. Two-Tier Memory Hierarchy (Instincts vs. Scratchpad)
- **Instincts Layer (L1 - Immutable Invariants)**: 프로젝트 코딩 스타일, 금지된 패키지, 보안 규칙 등 절대 삭제되지 않는 불변 제약.
- **Scratchpad Layer (L2 - FIFO with Summarization)**: 현재 하위 작업의 중간 관측치(Observation). 길이 임계치 초과 시 정보 엔트로피를 보존하는 요약 커널을 거쳐 압축됨.

---

## 3. 선행 런타임 하네스 대비 정량 비교 (Comparative Benchmark)

SWE-bench Lite 환경 기반 자율 버그 수정 성공률 및 토큰 효율성:

| 하네스 아키텍처 | SWE-bench 해결률 (Resolved) | 과제당 평균 토큰 소모량 | 도구 호출 실패율 | 샌드박스 안정성 |
| :--- | :---: | :---: | :---: | :---: |
| **Vanilla Tool Loop (ReAct)** | 18.2% | 145k 토큰 | 14.8% | ❌ 가드레일 없음 |
| **OpenHands / SWE-agent** | 27.5% | 110k 토큰 | 8.2% | ⚠️ 기본 도커 제한 |
| **Aider (Repo-Map)** | 26.3% | 75k 토큰 | 6.5% | ⚠️ 파일 편집 위주 |
| **ECC (Ours)** | **31.4%** | **52k 토큰 (최저)** | **2.1%** | **✅ 3단계 가드레일 & 롤백** |

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **과도한 안전 제약으로 인한 자율성 위축**: 엄격한 명령어 화이트리스트 적용 시, 복잡한 커스텀 빌드 도구나 비표준 패키지 매니저 실행이 차단되어 에이전트가 조기 포기할 수 있음.
2. **동적 스킬 로딩의 콜드 스타트 지연**: 스킬 레지스트리 검색 및 파싱 단계에서 미세한 디스패치 레이턴시가 발생함.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **자율 엔지니어링 에이전트의 런타임 벤치마킹**: 에이전트 연구에서 단순 모델 추론을 넘어 하네스 최적화가 전체 작업 완수율(Task Success Rate)에 미치는 기여도를 정량 분석하는 기준틀.
- **프로덕션 AI 에이전트의 보안 아키텍처**: 무인 자동화 환경에서 LLM의 잘못된 명령 실행을 원천 방어하는 프로덕션 레벨 가드레일 구현 표준.

