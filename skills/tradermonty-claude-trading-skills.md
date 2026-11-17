# 🛠️ Claude Trading Skills: Process Discipline as a Skill Graph — Declared Input Contracts, Fail-Closed Gates, and a Machine-Readable Verification Matrix

- **Repository**: [tradermonty/claude-trading-skills](https://github.com/tradermonty/claude-trading-skills)
- **Stars**: ⭐ 6,324 (forks 1,332, created 2025-10, last commit 2026-11-16)
- **Category**: `Skills` / Domain Skill Corpus & Decision-Gate Workflow
- **Organization**: tradermonty (MIT, personal project opened for reuse; English and Japanese documentation)
- **License / Runtime**: MIT / Python 3.9+ scripts invoked by skills, Markdown `SKILL.md` contracts, YAML state for the trade journal, plus `launchd` plists and shell scripts for local scheduling; optional FMP, FINVIZ Elite, and Alpaca integrations
- **Key Themes**: 74 indexed skills across 9 categories, 14 workflow manifests with typed steps and artifacts, integrations declared per skill, decision gates that fail closed on incomplete state, deterministic as-of evaluation, and an eight-axis verification matrix in which almost every axis is marked `not_verified`

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

투자 자동화의 실패는 대개 예측 실패가 아니라 **절차 실패**다. 손실 한도는 정해져 있는데 지키지 않고, 후보를 고르기 전에 위험을 확인하지 않으며, 기록이 없어서 개선이 축적되지 않는다. 이 저장소는 그 지점을 겨냥한다. 매수·매도 판단을 위임하는 것이 아니라, 판단의 **구조**를 스킬로 분해한다.

1. **순서의 붕괴(Order Collapse)**: 시장 국면 확인, 위험 예산, 후보 선별, 사이징, 진입 규율. 이 다섯 단계가 뒤섞이면 좋은 후보가 나쁜 시점에 큰 사이즈로 들어간다.
2. **입력의 암묵성(Implicit Inputs)**: 스킬이 무엇을 요구하는지 문서에 없으면, 에이전트는 없는 데이터로 그럴듯한 답을 만든다. "무료 CSV로도 돌아간다"는 주장과 "무엇이 없으면 무엇이 불가능하다"는 선언은 함께 있어야 한다.
3. **결측 상태의 처리(State Degradation)**: 기록이 일부만 있거나 서로 모순될 때, 조용히 건너뛰면 게이트는 형식이 된다.
4. **검증의 자기 신고(Self-Reported Quality)**: 스킬이 많아질수록 "검증됨"이라는 라벨의 근거가 흐려진다.

응답은 네 가지다. **명시적 입력 계약**, **게이트의 형식적 정의**, **결측 시 닫히는 기본값**, **정직한 검증 행렬**. 마지막 항목이 이 저장소의 가장 특이한 점이고, 역설적으로 가장 큰 약점을 스스로 드러낸다.

---

## 2. 아키텍처 및 수식화 (Architecture & Formulation)

### A. 스킬 인덱스: 선언된 계약

`skills-index.yaml`(2,454줄)은 각 스킬마다 범주, 상태, 검증 행렬, 운영 역할, 입력·출력, 통합 요구, 참여 워크플로를 선언한다. 74개 항목이 9개 범주(시장 국면 16, 전략 연구 15, 메타 9, 스윙 기회 7, 거래 계획 7, 고급 위성 7, 거래 기억 6, 배당 수익 5, 핵심 포트폴리오 2)에 배치되고, 상태는 프로덕션 62, 베타 12다. 운영 역할은 워크플로 단계 43, 독립 20, 연구 전용 6, 내부 구성요소 5다.

통합 선언은 요구 수준을 구분한다. 예를 들어 `backtest-expert`는 `user_input`(로컬 파일, 필수)만 요구하고, 스크리너 계열은 FMP 키를 요구하며, 포트폴리오 검토는 Alpaca를 요구하되 수동 CSV를 열화 경로로 명시한다. 이 구분이 있어야 "키가 없으면 무엇이 불가능한가"를 사용자가 계산할 수 있다.

무료 경로는 다섯 스킬로 명시된다: 시장 폭 분석(공개 CSV), 상승 추세 참여도(공개 CSV), 포지션 사이저(순수 계산), 거래 기억(로컬 YAML), 신호 사후 분석(프레임워크). 저장소는 여기에 "API 없음이 외부 데이터 없음은 아니다"라는 단서를 붙인다. 이 단서는 과장 방지 장치로 유효하다.

### B. 워크플로 매니페스트: 절차의 타입

워크플로는 기계가 읽는 YAML이다. `required_skills`, `optional_skills`, 산출물 목록(각각 `produced_by_step`, `required`, `downstream_hints`), 단계 목록(각 단계의 `skill`, `produces`, `decision_gate`, `optional`)으로 구성된다. 예를 들어 일일 시장 국면 워크플로는 4단계이고 15분 예상, 첫 세 단계는 데이터 산출, 4단계가 노출 자세 결정이다.

이 구조의 성질은 산출물 의존 그래프가 명시된다는 점이다. $A \to B$ 의존이 있으면 $A$의 산출물이 없을 때 $B$는 실행 불가 또는 열화 실행이다. 단계 번호가 곧 위상 정렬이므로, 매니페스트는 DAG의 한 표현이다.

### C. 게이트의 형식: 전함수와 결측 처리

`drawdown-circuit-breaker`는 게이트의 모범 사례다. 상태 디렉터리의 YAML 원장에서 실현 손익을 읽고 규칙을 적용한다. 기본 규칙은 일일 손실 2.0%, 연속 손실 2건(24시간 쿨다운), 주간 낙폭 5.0%, 월간 낙폭 8.0%이고, 경계는 모두 `America/New_York` 기준이며 `--as-of`로 결정론적 평가가 가능하다.

판정은 값과 데이터 품질의 곱이다.

$$
G : \mathcal{S} \to \lbrace \text{ALLOWED}, \text{COOLDOWN}, \text{HALTED} \rbrace \times \lbrace \text{OK}, \text{EMPTY}, \text{PARTIAL} \rbrace
$$

핵심은 실패 방향이다. 상태가 존재하지만 일부 원장 이벤트를 건너뛰어야 하거나 값이 충돌하면 `PARTIAL` + `HALTED`로 **닫힌다**. 반면 상태 디렉터리가 없거나 비어 있으면 `ALLOWED` + `EMPTY_STATE`로 **열린다**. 이 예외는 의도적이고 문서에 근거가 있다. 신규 사용자를 이력 부재로 막지 않기 위해서다. 즉 게이트는 기본적으로 fail-closed이되, "이력 없음"이라는 특정 결측만 fail-open으로 분류한다. 결측의 종류를 구분하지 않는 fail-closed는 사용 불가능한 시스템을 만들고, 구분하는 fail-closed는 신뢰할 수 있는 시스템을 만든다.

중복 처리도 같은 원리다. 같은 `thesis_id`를 가진 유효 파일이 둘 이상이면 그 그룹 전체를 손익·연속 손실 계산에서 제외하고, 실제 경로를 모두 보고하며, 복구될 때까지 `PARTIAL` + `HALTED`를 유지한다. 잘못된 값을 고르는 대신 판정을 멈춘다.

### D. 자세의 격자와 만남 연산

노출 자세는 `allow`, `restrict`, `cash-priority`의 세 단계로 표현되고, 시장 게이트(노출 코치)와 계좌 게이트(서킷 브레이커)가 각각 자세를 낸다. 두 게이트의 결합은 **만남**(meet)이다.

$$
\text{posture} = \bigwedge_{g \in \text{gates}} \text{posture}_g, \qquad \text{cash-priority} \le \text{restrict} \le \text{allow}
$$

각 게이트가 제한만 할 수 있다는 성질(축소 사상)과 결합이 만남이라는 성질이 함께 성립하면, 게이트를 추가할수록 자세는 같거나 더 보수적이 된다. 게이트 추가가 위험을 늘릴 수 없다는 단조성은 안전 규칙 집합의 필수 성질이다.

신뢰도도 함께 내려간다. 예시 화면은 핵심 입력이 빠진 상태에서 노출 상한 36%, `REDUCE_ONLY`, 신뢰도 `LOW`를 함께 표시한다. 즉 **결측은 판정을 막는 대신 판정의 확신을 낮춘다**는 두 번째 전략이 있다. 두 전략(닫기 대 확신 낮추기)이 한 시스템에 공존하며, 어느 쪽을 쓸지는 결측의 성격에 달렸다.

### E. 검증 행렬: 여덟 축의 정직한 공백

인덱스의 각 스킬에는 8개 축의 검증 상태가 붙는다: `instruction_contract`, `unit_tests`, `workflow_contract`, `end_to_end_replay`, `data_provenance`, `financial_logic_review`, `empirical_validation`, `security_review`. 값은 `passed`, `not_verified`, `not_applicable` 중 하나다.

| 축 | passed | not_verified | not_applicable | 미기재 |
| :--- | ---: | ---: | ---: | ---: |
| instruction_contract | 1 | 62 | 0 | 11 |
| unit_tests | 13 | 47 | 3 | 11 |
| workflow_contract | 5 | 32 | 26 | 11 |
| end_to_end_replay | 5 | 33 | 25 | 11 |
| data_provenance | 0 | 62 | 1 | 11 |
| financial_logic_review | 1 | 61 | 1 | 11 |
| empirical_validation | 0 | 54 | 9 | 11 |
| security_review | 23 | 40 | 0 | 11 |

총 `passed`는 48건이다. 적용 가능 셀(총 592개에서 해당 없음 65개와 미기재 88개를 뺀 약 439개) 기준으로

$$
\frac{\sum_{a} \text{passed}_a}{\lvert \text{applicable cells} \rvert} \approx \frac{48}{439} \approx 0.11
$$

즉 **검증 표시의 11% 정도만 채워져 있고, 재무 로직 검토 1건, 데이터 출처 검증 0건, 실증 검증 0건**이다. 트레이딩 도메인에서 이 세 축이 비어 있다는 것은 중대한 사실이다. 이 저장소는 그 사실을 숨기지 않고 기계 판독 가능한 형태로 공개한다. 이 정직성이 이 저장소의 가장 값진 자산이며, 동시에 사용자가 반드시 읽어야 할 경고다.

---

## 3. 구조적 비교 (Comparative Positioning)

| 항목 | 단일 프롬프트 트레이딩 봇 | 상용 스크리너 | 백테스트 라이브러리 | Anthropic 예제 스킬 | Claude Trading Skills |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 절차의 순서 보장 | 없음 | 부분 | 없음 | 없음 | 워크플로 매니페스트 |
| 입력 계약 선언 | 없음 | 없음 | 부분 | 부분 | 스킬별 통합 선언 |
| 결측 시 동작 | 조용히 진행 | 조용히 진행 | 예외 | 없음 | fail-closed + 예외 규정 |
| 게이트의 형식 정의 | 없음 | 없음 | 없음 | 없음 | 전함수 + 자세 격자 |
| 검증 근거의 공개 | 없음 | 벤더 주장 | 테스트 | 예시 | 8축 행렬(대부분 미검증) |
| 스킬 수 | 1 | — | — | 소수 | 74 |
| 로컬·무료 경로 | 있음 | 없음 | 있음 | 있음 | 명시된 5스킬 경로 |

`skills/` 안에서 이 저장소의 자리는 "규격 → 방법론 → 도메인 번들 → 산출물 파이프라인 → **절차 그래프**"다. 지식이 아니라 **순서와 조건**을 배포하는 코퍼스이고, 그래서 규모가 커질수록 가치는 게이트와 계약에서 나온다.

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **검증의 공백**: 8축 중 재무 로직 검토 1건, 데이터 출처 0건, 실증 검증 0건이다. 스킬의 룰이 실제 시장에서 옳은지에 대한 증거가 저장소 안에 없다. 프로덕션 라벨은 성숙도를 뜻하지 않는다.
2. **인덱스와 파일의 불일치**: `SKILL.md` 파일은 94개인데 인덱스 항목은 74개다. 20개가 인덱스 밖에 있고, 그중 일부는 문서·실험 단계일 수 있지만 그 구분이 기계적으로 드러나지 않는다.
3. **지역·규제 편향**: 경계 시간대가 미국 동부이고, 세금·계좌 관련 스킬은 미국 제도를 전제한다. 한국·일본 사용자는 시장 시간과 규제 해석을 직접 바꿔야 한다.
4. **데이터 의존의 계단**: 무료 경로는 CSV와 스크린샷을 요구하고, 스크리너는 유료 API를 요구한다. "무료로 시작"과 "실전 운용" 사이에 데이터 품질의 단절이 있고, 그 단절이 판단 품질에 미치는 영향이 측정되지 않는다.
5. **스킬 수의 역효과**: 74개 스킬은 발견 비용을 만든다. 에이전트가 5개 중 1개를 고르는 것과 74개 중 1개를 고르는 것은 다른 문제이고, 이 저장소는 라우팅 정확도를 측정하지 않는다.
6. **게이트의 우회 가능성**: 게이트는 스킬로 구현된 권고이고 브로커 측 차단이 아니다. 사용자가 게이트를 호출하지 않으면 아무 일도 일어나지 않는다. 강제 계층이 없다는 점이 문서에 명시되어 있지만, 실무에서는 이것이 가장 흔한 실패다.
7. **표본 편향**: 사후 분석 스킬은 기록된 거래에만 작동한다. 기록되지 않은 판단(진입하지 않은 후보, 중도 포기)은 학습 대상에서 빠지므로, 개선이 생존 편향 위에 쌓인다.
8. **언어·문서 표면**: 영어·일본어 문서가 병행되고 워크플로 YAML에 두 언어 필드가 있다. 유지 비용이 두 배가 되고, 한쪽만 갱신되면 계약이 어긋난다.
9. **자동화 표면의 취약성**: `launchd`와 셸 스크립트는 macOS 로컬 실행을 전제한다. 스케줄 실행의 실패가 조용하면 게이트는 그날 작동하지 않는다.
10. **성과 주장의 부재**: 예측력이나 수익률을 주장하지 않는 것은 정직하지만, 그렇다면 사용자가 얻는 것은 **일관된 절차**뿐이다. 절차가 성과를 개선한다는 증거도 아직 없다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **결측의 종류별 처리**: "이력 없음"은 열고 "이력 불완전"은 닫는 구분은 fail-closed 설계의 실용적 정교화다. 결측을 한 덩어리로 다루는 시스템은 신규 사용자를 막거나 오염된 상태를 통과시킨다.
- **게이트를 전함수로**: 판정과 데이터 품질을 곱으로 돌려주는 형식은 게이트가 침묵할 수 없게 만든다. 침묵 대신 `PARTIAL`이라는 값을 갖게 하는 것만으로 감사 가능성이 크게 올라간다.
- **자세 격자의 만남 연산**: 제한만 가능한 게이트들을 만남으로 결합하면 게이트 추가가 위험을 늘릴 수 없다는 단조성이 자동으로 성립한다. 여러 안전 규칙을 합치는 모든 설계에 적용된다.
- **검증 행렬의 공개**: 축별로 `passed`/`not_verified`/`not_applicable`을 선언하는 방식은 스킬 코퍼스의 신뢰도를 사용자가 계산하게 만든다. 채워진 비율이 11%라는 사실을 숨기지 않는 것이 오히려 재사용의 근거가 된다.
- **산출물 의존의 명시**: 워크플로 매니페스트에서 산출물의 생산 단계와 소비자를 적으면, 실행 계획과 실패 복구가 데이터로 다뤄진다. 산문 절차서와의 결정적 차이다.
- **후속 연구 방향**: (i) 게이트가 실제로 손실을 줄이는지의 전향적 검증, (ii) 스킬 수와 라우팅 정확도의 관계, (iii) 기록되지 않은 판단의 수집 방법, (iv) 데이터 품질 계단이 판단 품질에 미치는 영향, (v) 재무 로직의 형식 검증(속성 기반 테스트) 도입.
