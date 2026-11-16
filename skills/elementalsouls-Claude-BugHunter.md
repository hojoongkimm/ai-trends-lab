# 🛠️ Claude-BugHunter: Domain Expertise as a Skill Bundle — 83 Auto-Routed Skills over 681 Disclosed-Report Patterns, a Deterministic Scope Guard, and an Ablation That Refuses to Flatter Itself

- **Repository**: [elementalsouls/Claude-BugHunter](https://github.com/elementalsouls/Claude-BugHunter)
- **Stars**: ⭐ 6,400+ (forks 960+, created 2026-05, 6 open issues)
- **Category**: `Skills` / Domain Skill Bundle & Offensive-Security Workflow
- **Organization**: Independent (elementalsouls, GenAI Security Research)
- **License / Runtime**: MIT for code (`*.py`, `*.sh`) and CC BY 4.0 for content (skills, wordlists, rubrics, docs) / Python 3 standard library engine (about 3.2k LOC under `engine/` and `cbh/`, about 9.5k LOC of Python in the repository including the eval harness), 83 `SKILL.md` files totaling 1.69 MB with about 52 KB of front-matter descriptions, 15 slash commands, 5 harness install targets (Claude Code, OpenCode, Codex CLI, Hermes, AntiGravity)
- **Key Themes**: description-triggered skill routing across 58 `hunt-*` class skills, a 7-Question conjunction gate with four outcomes, deterministic deny-wins scope enforcement in code rather than in prose, `$0` recon with LLM only on opt-in hunt, cross-engagement negative memory with a dead-class threshold, and a self-graded skills-on/off ablation whose published baseline shows no capability gain

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

보안 도메인에서 에이전트의 실패는 "무엇을 아는가"보다 **무엇을 언제 꺼내는가**와 **무엇을 제출하지 않는가**에서 발생한다. 이 저장소가 겨냥하는 병목은 세 가지다.

1. **검색·디스패치의 비용(Retrieval Cost)**: 681개 공개 보고서에서 정리한 24개 취약점 클래스 지식은 컨텍스트에 통째로 들어갈 수 없다. 전부 넣으면 토큰이 터지고, 필요한 것만 넣으려면 "이 엔드포인트는 어느 클래스인가"를 먼저 판정해야 한다. 지식의 양이 아니라 지식의 색인 방식이 성능을 결정한다.
2. **오탐의 경제학(False-Positive Economics)**: 버그바운티에서 제출물의 가치는 해결률이 아니라 **유효성 비율**, 즉 validity ratio로 측정된다. 정보성(informative) 제출은 중립이고 N/A는 손실이다. 따라서 좋은 조언은 "더 많이 찾으라"가 아니라 "제출하기 전에 죽여라"가 되며, 이는 판정 함수의 문제로 바뀐다.
3. **범위와 권한(Scope & Authorization)**: 대상 범위는 법적·계약적 제약이다. 모델이 산문 지시를 따르기를 기대하는 방식으로는 보장되지 않고, 엔진이 무인 실행을 전제한다면 범위 검사는 코드에 있어야 한다.

이 세 병목은 각각 **색인**, **판정**, **정책**이라는 서로 다른 층의 해법을 요구한다. 이 번들의 구조는 그 분리를 그대로 반영한다. 스킬 83개가 색인이고, 7-Question Gate가 판정이며, `scope.py`와 `engine.py`가 정책이다. 흥미로운 점은 저자가 세 층의 효과를 측정하려 했고, **측정 결과가 자기 주장과 어긋났을 때 그대로 공개했다**는 것이다. 2절 G와 4절 1항이 그 부분이다.

---

## 2. 아키텍처 및 수학적 정식화 (Architecture & Formulation)

### A. 코퍼스 구조: 스킬은 함수가 아니라 색인 단위

구성은 계층적이다. 사고 층(`bb-methodology`, `redteam-mindset`), 웹 취약점 층(58개 `hunt-*`), 기업 인프라 층(10개: M365/Entra, Okta, vCenter, SSL-VPN, SharePoint, cloud IAM, APK 등), 그리고 마감 층(`triage-validation`, `evidence-hygiene`, `report-writing`, `bugcrowd-reporting`)이다. 분류는 55개 자체 작성, 20개 커뮤니티(v3), 8개 상류 저장소에서 가져온 것이다.

스킬은 이름으로 호출되지 않고 **설명(frontmatter `description`)으로 자동 발동**한다. 따라서 상주 비용과 호출 비용이 분리된다. 설명 색인 $D$, 클래스 $c$의 본문 $B_c$, 사전 분포 $p(c)$에 대해 한 세션의 기대 컨텍스트 비용은 다음과 같다.

$$
\mathbb{E}[C] = \lvert D \rvert + \sum_{c} p(c)\lvert B_c \rvert
$$

실측 가능한 값은 $\lvert D \rvert \approx 52\mathrm{KB}$(83개 설명 합계)와 $\sum_c \lvert B_c \rvert \approx 1.69\mathrm{MB}$(전체 본문)다. 전체를 넣는 것이 불가능하고, 색인만으로는 본문이 없다. 이 구조가 성립하려면 모델의 설명 매칭이 정확해야 하며, 저장소에는 임베딩 검색이나 결정론적 라우터가 없다. 디스패치 품질이 곧 모델의 판단이라는 뜻이고, 4절 2항의 한계가 여기서 나온다.

부수 효과로 스킬 파일 자체가 지식 베이스의 단위가 된다. 각 `hunt-*`는 탐지 패턴, 우회 표, 체인 템플릿, 공개 보고서 인용을 담고, 문서에 따르면 433개 패턴이 개별 인용되어 감사 가능하다. 지식의 출처를 파일 단위로 추적할 수 있게 만든 것이 이 코퍼스의 실질적 자산이다.

### B. 판정 함수: 7-Question Gate와 4개 출구

검증은 7개 질문의 논리곱이다. 하나라도 거짓이면 그 발견은 죽는다.

$$
\mathrm{submit}(f) \iff Q_1(f) \wedge Q_2(f) \wedge \cdots \wedge Q_7(f)
$$

질문은 순서대로 재현성($Q_1$: 실제 HTTP 요청으로 단계별 서술이 가능한가), 프로그램 수용 범위($Q_2$), 자산 범위($Q_3$), 권한 현실성($Q_4$), 기지 여부($Q_5$), 영향 증명($Q_6$), 금지 클래스($Q_7$)를 본다. 이어 4개 제출 전 게이트(현실성·영향·중복·보고서 품질)가 순차로 돌고, 결과는 PASS·KILL·CHAIN-REQUIRED·DOWNGRADE 네 출구로 갈린다. 게이트의 목적 함수는 명시적이다.

$$
u(f) = v(f)\mathbb{1}\bigl[\mathrm{submit}(f)\bigr] - c_{\text{test}} - c_{\text{submit}} - \lambda\mathbb{1}\bigl[\mathrm{N/A}(f)\bigr], \qquad \lambda \gt 0
$$

$Q_6$이 "기술적으로 가능"에 머물면 죽이지 않고 강등하는 비대칭은, 증거가 약한 발견을 버리지 않으면서 과대평가도 하지 않으려는 절충이다.

가장 흥미로운 조각은 **계층 순서 함정**, 즉 layer-ordering trap이다. 인증 우회를 주장하는 흔한 오추론은 "인증 없이 보냈더니 검증 오류가 돌아왔으니 인증을 통과했다"인데, 이는 인증 미들웨어가 입력 처리보다 먼저 실행될 때만 성립한다. 요청 본문 파서나 스키마 필터가 앞에 있으면 잘못된 본문은 인증에 닿기 전에 거부되고, 두 경우의 응답은 구별되지 않는다. 처방은 **잘 형성된 최소 본문으로 다시 보내는 것**이다. 관측이 두 가설에 의해 같은 확률로 설명될 때, 관측을 늘리지 않고 가설을 구분하려는 시도가 오탐의 최대 원천이라는 것을 정확히 짚는다.

### C. 정책을 코드로: deny-wins 결정론적 범위 검사

`engine/scope.py`는 허용목록을 다섯 형태(apex, 와일드카드, 정확한 호스트, CIDR, `re:` 정규식)로 파싱하고, 호스트 $h$에 대해 다음을 계산한다.

$$
\mathrm{allowed}(h) = \neg \Bigl(\bigvee_{o \in O} m(o, h)\Bigr) \wedge \Bigl(\bigvee_{i \in I} m(i, h)\Bigr)
$$

$O$는 제외 목록, $I$는 포함 목록, $m$은 패턴 매칭이다. 제외가 이기고(deny wins), 어디에도 맞지 않으면 범위 밖이다(default deny). 이 술어는 정찰 단계와 헌트 단계 양쪽에서 호출되고, 모든 에이전트 호출 전에 검사된다. 산문이 아니라 코드라는 점이 핵심이다. 무인 실행에서 모델의 준수를 기대하지 않고, 모델이 틀릴 수 있는 지점을 술어 하나로 좁힌다.

### D. 엔진의 비용 구조: 결정론 우선, 에이전트는 옵트인

기본 실행은 에이전트를 쓰지 않는다. 서비스·기술 핑거프린트, JS 번들 엔드포인트 마이닝, JS 비밀 스캔(값은 마스킹), 그리고 두 독립 출처(`gau`의 과거 수동 수집과 `katana`의 실시간 크롤)에서 뽑은 파라미터를 모아 클래스별로 분류하고, `skill_map.py`가 `(엔드포인트 또는 파라미터, 클래스)`를 **실제로 설치된** `hunt-*` 스킬과 curl 시작 프로브에 대응시킨다. 산출물은 `arsenal.md`이고, 실행은 거기서 멈춘다.

$$
C_{\text{map}} = O(\lvert \text{hosts} \rvert \cdot \lvert \text{params} \rvert)\quad \text{의 결정론 연산}, \qquad C_{\text{hunt}} = O(\lvert \text{candidates} \rvert)\quad \text{의 에이전트 호출}
$$

에이전트는 순위가 매겨진 후보에만 붙고, 기본은 읽기 전용이며 상태 변경 PoC는 `--allow-intrusive`로 명시해야 한다. 병렬 헌트는 `--parallel`, 상한은 `--max-hunts`로 제한된다.

기억 층은 탐색-활용 계산을 파일로 한다. `memory.py`는 확정 발견, 미확정 부정 기록, 호스트별 롤업을 JSONL로 쌓고, 같은 기술 스택에서 같은 클래스가 **5회** 부정되면 그 조합을 "죽은 클래스"로 표시해 이후 호출을 건너뛴다(`DEAD_CLASS_THRESHOLD = 5`). 파일은 10 MB에서 회전하고 최근 3개를 보존한다. 이것은 "이 스택에서 이 클래스는 수확이 없다"는 사실을 세션 밖에 저장해 재시도를 줄이는, 소박하지만 올바른 형태의 학습이다.

### E. 명령 표면과 하네스 이식

15개 슬래시 명령이 6단계 흐름을 나눠 맡는다(`/hunt`, `/recon`, `/scope`, `/surface`, `/triage`, `/validate`, `/chain`, `/report`, `/autopilot`, `/intel`, `/pickup`, `/remember`, `/memory-gc`, `/token-scan`, `/web3-audit`). 지식 층은 5개 하네스(Claude Code, OpenCode, Codex CLI, Hermes, AntiGravity)에 같은 `SKILL.md` 형식으로 복사되고, 명령과 `/hunt` 엔진은 Claude Code 전용으로 남는다. 이식 범위를 의도적으로 비대칭으로 둔 것은 정직한 선택이다. 슬래시 명령과 훅 기반 오케스트레이션은 호스트 API에 의존하므로, 흉내내면 조용히 다른 동작을 만든다.

`/hunt`는 첫 턴에 참여 프레임(권한·범위·재현 가능한 산출물)을 한 번 선언하고, SOW 문서 자체는 붙여넣지 말라고 지시한다. 민감한 고객 문서를 수집·저장하지 않으면서 범위는 `/scope`와 게이트로 강제한다는 구분이며, 권한의 증명을 수집하지 않고 증명되었다고 가정한다. 이 가정의 위험은 4절 4항에서 다룬다.

### F. 자격증명이 아니라 지식: 실행 환경의 경계

이 번들은 도구를 실행하지 않고 **판단과 절차를 공급**한다. 실제 요청은 에이전트가 `curl`이나 Burp MCP로 보낸다. 따라서 경계는 저장소가 아니라 호스트에 있다. 그 결과 두 가지가 따라온다. (i) 산출물의 품질이 호스트의 도구 권한에 그대로 종속된다. (ii) 모델 제공자의 실시간 사이버 안전장치가 개입하면, **권한이 있는 작업도 거부되거나 다른 모델로 폴백**될 수 있다. 저장소는 이 현상을 숨기지 않고, Opus 5의 고위험 사이버 요청이 거부 대신 Opus 4.8로 폴백된다는 점과 Cyber Verification Program 등록 경로를 문서에 적는다. 스킬 번들이 런타임 정책에 의해 매개되는 능력이라는 사실을 인정하는 서술이다.

### G. 증거: 자기 채점 오라클을 붙인 ablation

`eval/`은 주장을 측정으로 바꾸려는 시도다. 구조는 단순하고 옳다. 표적을 초기화하고(`docker restart`), 헤드리스 에이전트(`claude -p`)에게 스킬과 Burp MCP를 주고, 표적이 스스로 채점하는 오라클(Juice Shop의 `/api/Challenges` 해결 플래그, 또는 PortSwigger 랩의 `is-solved` 위젯)을 폴링해 `{solved, turns, cost, tokens}`를 기록한다. 조건 간 차이는 스킬만 끄는 것으로 만든다(`--disable-slash-commands`).

$$
\Delta_{\text{skill}} = \mu_{\text{skills-on}} - \mu_{\text{skills-off}}
$$

공개된 기준선(2026-08-25, `claude-opus-4-8`, 3개 챌린지)은 다음과 같다.

| 조건 | 해결 | 해결률 | 중앙 턴 수 | 중앙 비용(USD) |
| :--- | ---: | ---: | ---: | ---: |
| skills-off | 3/3 | 100% | 2 | 0.178 |
| skills-on | 3/3 | 100% | 3 | 0.254 |
| 차이 | +0 | +0% | +1 | +0.076 |

즉 $\Delta_{\text{solved}} = 0$, $\Delta_{\text{turns}} = +1$, $\Delta_{\text{cost}} = +0.076$이다. 저자의 해석도 정확하다. 암기된 표적에서는 천장 효과가 생기고, 스킬 컨텍스트는 순수한 오버헤드다. 이 결과는 v0를 "파이프라인 증명 + 자율성·비용 측정"으로 한정하고, 진짜 스킬 델타는 덜 암기된 표적(PortSwigger)에서 재야 한다고 못박는다. 같은 결론이 엔진 코드에도 반영되어 있다. `engine/agent.py`는 스킬을 **기본으로 끄고**, 그 이유를 "eval에서 능력 상승은 거의 0인데 에이전트당 12–15k 토큰을 쓴다"로 적는다. 측정이 설계를 바꾼 드문 사례다.

---

## 3. 구조적 비교 (Comparative Positioning)

| 항목 | 단일 보안 프롬프트 | 큐레이션 링크 모음 | 스킬 번들(이 저장소) | 사람 플레이북 |
| :--- | :--- | :--- | :--- | :--- |
| 지식 단위 | 한 문단 | 북마크 | 83개 `SKILL.md` + 인용 | 문서·경험 |
| 발동 방식 | 항상 로드 | 사람이 찾음 | 설명 기반 자동 | 사람이 판단 |
| 범위 강제 | 없음 | 없음 | 코드(deny-wins) + 게이트 | 계약서·사람 |
| 오탐 통제 | 없음 | 없음 | 7-Question Gate, 4개 출구 | 리뷰 문화 |
| 상주 토큰 | 작음 | 0 | 약 52 KB 색인 + 선택 본문 | 0 |
| 실행 비용 | 낮음 | 0 | 맵은 무료, 헌트는 옵트인 | 사람 시간 |
| 효과 증거 | 없음 | 없음 | 자기 채점 ablation(공개, 음성) | 없음 |

같은 `skills` 카테고리의 다른 문서들과 비교하면 위치가 분명해진다. Anthropic의 `skills`는 규격을, Addy Osmani의 팩은 엔지니어링 규율을, Superpowers는 방법론을 공급한다. 이 번들은 **측정 대상이 명확한 도메인 지식**을 공급하고, 그 측정에서 자기에게 불리한 결과를 먼저 공개한다. 코퍼스의 크기(1.69 MB)와 색인(52 KB)의 비율, 그리고 색인 정확도가 성능을 좌우한다는 구조는 다른 대형 스킬 팩에도 그대로 적용되는 관찰이다.

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **자기 평가가 능력 상승을 확인하지 못했다**: 공개 기준선은 천장 효과로 델타 0을 보였고, 비용은 오히려 늘었다. 진짜 델타를 재려는 PortSwigger 계층은 계정과 수동 랩 실행을 요구해 반자동으로 남아 있다. 즉 "이 번들이 실력을 올린다"는 주장은 아직 측정되지 않은 상태이며, 저장소도 그렇게 적는다.
2. **디스패치가 모델의 설명 매칭에 의존**: 83개 설명 중 무엇이 발동할지는 모델 판단이다. 임베딩 검색도, 결정론적 라우터도 없으므로, 유사 클래스(예: 인증 우회와 세션, 캐시 포이즌과 요청 스머글링)에서 오발동·미발동이 생긴다. 오발동은 토큰을, 미발동은 커버리지를 잃는다.
3. **산문 규범과 코드 강제의 간극**: 범위 검사는 엔진·`cbh` 경로에서만 코드다. 스킬을 대화형으로 쓸 때의 범위 준수는 여전히 모델의 순응에 달려 있고, 게이트도 산문 체크리스트다. 정책의 강제 범위가 실행 경로에 따라 달라진다.
4. **권한은 가정되고 검증되지 않는다**: `/hunt`는 권한을 주장으로 받아들이고 SOW 수집을 거부한다(프라이버시상 합리적). 그 결과 오용의 책임은 전적으로 운영자에게 있고, 번들은 그것을 기술적으로 막지 않는다. 이중용도 도구의 일반적 한계이며, 제공자 측 안전장치(CVP, 모델 폴백)가 사실상의 외부 게이트가 된다.
5. **품질 분산**: 83개 중 55개 자체 작성, 20개 커뮤니티, 8개 상류 이식이다. 스킬별 평가는 없고 CI는 하네스 동작만 점검한다. 어떤 클래스는 인용과 표가 촘촘하고 어떤 클래스는 얇다.
6. **문서·수치의 드리프트**: 저장소 설명은 82개 스킬이라 적고 README는 83개라 적으며, 워크플로를 6단계라 부르면서 단계 이름은 5개다. 681개 패턴 중 433개만 개별 인용이라는 비율도 스스로 밝힌다. 지식 코퍼스에서 수치 신뢰도는 곧 감사 가능성이다.
7. **유지비의 시간 의존성**: 기업 인프라 CVE 체인은 분기 갱신이 로드맵에 있고, 갱신이 밀리면 지식이 조용히 낡는다. 공개 보고서 기반 패턴도 프로그램 규칙 변경을 따라가지 못한다.
8. **ablation의 교란**: 스킬을 끄는 수단이 `--disable-slash-commands`라서, 스킬과 슬래시 명령이 함께 사라진다. 조건 간 차이가 순수한 지식 효과인지 워크플로 효과인지 분리되지 않는다.
9. **비용의 비선형성**: 후보 수에 비례해 에이전트 호출이 늘고, 병렬도와 상한을 사람이 정해야 한다. 토큰 예산 관리가 운영자의 몫으로 남는다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **도메인 지식을 색인 가능한 단위로 만드는 설계**: 설명(frontmatter)과 본문을 분리해 상주 비용과 호출 비용을 나누는 구조는, 대형 지식 팩이 컨텍스트 예산 안에서 작동하는 일반 조건을 보여준다. 색인 크기, 본문 크기, 발동 정확도의 세 변수로 스킬 팩을 비교하는 벤치마크가 자연스럽다.
- **판정을 논리곱 게이트로 만드는 것**: "제출 전에 죽여라"는 규율을 7개 명제의 논리곱과 4개 출구로 바꾸면, 규율이 측정 가능한 절차가 된다. 이 형태는 보안을 넘어 논문 심사, 특허 출원, 의료 보고처럼 **오탐 비용이 큰 제출** 전반에 이식된다.
- **정책을 프롬프트 밖으로**: deny-wins 허용목록을 코드에 두는 선택은 에이전트 안전 연구의 핵심 패턴(정책 집행과 판단의 분리)과 일치한다. 판단은 모델이, 집행은 술어가 맡고, 술어는 단위 테스트가 가능하다.
- **음성 결과의 가치**: 천장 효과와 비용 증가를 그대로 공개한 기준선은, 스킬 팩 논의에서 흔히 빠지는 "왜 이게 도움이 되는가"를 데이터로 남긴다. 동일 하네스로 다른 팩을 재면 팩 간 비교가 가능해진다.
- **탐색-활용의 파일 기반 기억**: 부정 기록과 죽은 클래스 임계값은 단순하지만 재현 가능한 형태의 경험 학습이다. 임계값 5를 스택별로 캘리브레이션하는 문제, 부정 기록이 오래된 표적에 대해 거짓이 되는 문제는 그 자체로 연구 질문이다.
- **후속 연구 방향**: (i) 스킬별 개별 ablation과 발동 정확도 측정, (ii) 설명 기반 라우팅과 검색 기반 라우팅의 비용·정확도 비교, (iii) 게이트 임계값과 유효성 비율의 관계(캘리브레이션), (iv) 지식 코퍼스의 시간에 따른 부패율과 갱신 전략, (v) 모델 제공자 안전장치가 도구 능력에 미치는 영향을 측정하는 표준 방법.
