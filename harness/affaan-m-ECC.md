# ⚡ ECC: A Harness Operating System — 72 Agents and 327 Skills Behind a Seven-Stage Loop, Confidence-Scored Instincts, and a Static Auditor for the Agent's Own Configuration

- **Repository**: [affaan-m/ECC](https://github.com/affaan-m/ECC)
- **Stars**: ⭐ 306,498 (forks 45,120, created 2026-01-18, 169 open issues, last push 2026-11-17)
- **Category**: `Harness` / Agent Runtime System & Engineering Process Substrate
- **Organization**: Independent (affaan-m), MIT
- **License / Runtime**: MIT / Node.js 18+ for the universal package, Claude Code CLI 2.1+ for the plugin path; release 2.2.3 (2026-10-01); 72 agents, 327 skill directories, 98 maintained command shims, 28 hook entries across 8 lifecycle events, rule sets for 21 languages plus `common`
- **Key Themes**: a seven-stage engineering loop installed as runtime rather than prompt, three parallel component surfaces (agents, skills, commands) with an explicit migration order, a hook-observed instinct memory scored in 0.3–0.9 and scoped per project, a context budget the README states as numbers rather than intentions, and AgentShield, a 102-rule static auditor pointed at the agent's own hooks, MCP config, permissions and secrets

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

프런티어 코딩 에이전트의 실패는 대부분 모델 능력이 아니라 **절차가 실행되지 않는 것**에서 온다. 같은 모델이 한 세션에서는 계획을 세우고 테스트를 먼저 쓰지만, 다음 세션에서는 곧바로 코드를 쓰고, 리뷰는 자기가 쓴 코드를 자기가 읽는 방식으로 끝난다. 프롬프트로 절차를 요구하는 방식이 실패하는 이유는 절차가 **확률적 준수**에 맡겨지기 때문이다. 이 저장소가 겨냥하는 병목은 네 가지다.

1. **절차 준수의 확률성(Process Compliance)**: 스킬은 모델의 판단으로 발동하므로 준수율이 100%가 아니다. ECC는 같은 문제를 다른 층에서 관찰한다. 스킬 기반 관찰은 50–80%만 발화하지만 훅은 100% 발화한다는 관측이 그대로 저장소의 설계 근거로 적혀 있다. 절차를 지키게 하려면 절차를 **이벤트에 묶어야** 한다.
2. **세션 경계에서의 기억 소멸(Memory Loss at the Boundary)**: 압축과 세션 종료는 곧 배운 것의 소실이다. 사람 팀은 회고와 위키로 이를 보완하지만 에이전트에게는 그 층이 없었다. 여기서 필요한 것은 대화 로그가 아니라 **원자 명제와 신뢰도**를 가진 기억이다.
3. **구성 표면의 공격성(Configuration as Attack Surface)**: 훅은 셸을 실행하고 MCP 서버는 자격증명을 쥐며 프로젝트 지시문은 컨텍스트에 들어간다. 셋 다 실행 가능한 구성이다. 그런데 에이전트 설정을 감사하는 도구는 오랫동안 없었다.
4. **지식량 대 예산(Knowledge Mass vs. Budget)**: 327개 스킬과 21개 언어 규칙을 모두 실을 수는 없다. 무엇을 상주시키고 무엇을 색인만 둘지가 성능을 결정한다.

응답의 형태가 흥미롭다. ECC는 이 네 병목을 각각 **루프**, **기억**, **감사기**, **예산 규칙**이라는 서로 다른 층으로 분리하고, 각 층을 호스트의 실제 훅·플러그인 API에 붙인다. 슬로건도 그 분리를 그대로 말한다. Optimize the context window. Persist everything else.

---

## 2. 아키텍처 및 수학적 정식화 (Architecture & Formulation)

### A. 일곱 단계 루프: 절차를 상태 기계로

README가 제시하는 루프는 `plan → test → implement → review → verify → remember → improve` 일곱 단계다. 중요한 것은 단계 이름이 아니라 **각 단계의 소유자와 산출물**이 고정되어 있다는 점이다.

| 단계 | 소유 표면 | 산출물 | 다음 단계로 넘어가는 조건 |
| :--- | :--- | :--- | :--- |
| plan | `planner` 에이전트, `/ecc:plan` | 구현 청사진 | 인터페이스와 작업 분해가 문서에 존재 |
| test | `tdd-guide`, `tdd-workflow` | 실패하는 테스트 | 테스트가 실제로 실패함을 관측 |
| implement | 기본 세션 | 최소 구현 | 테스트가 통과 |
| review | `code-reviewer` (새 컨텍스트) | 심각도별 이슈 | 치명 결함 0 |
| verify | `e2e-runner`, `verification-loop` | 실행 증거 | 80% 이상 커버리지 |
| remember | `continuous-learning-v2` | instinct | 관측이 원자 명제로 승격 |
| improve | `/evolve`, `skill-create` | 새 스킬·커맨드 | 클러스터가 재사용 단위가 됨 |

형식적으로 각 단계 $s$는 사전조건 $P(s)$와 사후조건 $Q(s)$를 갖고, 루프는 사후조건이 만족될 때만 다음 단계로 진행하는 전이 계다.

$$
\mathrm{advance}(s \to s+1) \iff Q(s)\left(A_s, x_s\right) = \top
$$

여기서 $A_s$는 단계 소유 에이전트, $x_s$는 그 단계의 산출물이다. 가장 값싼 설계 결정은 **review의 컨텍스트 분리**다. 리뷰어가 작성자와 같은 컨텍스트를 공유하면 리뷰는 자기 확신의 재확인이 된다. ECC는 `code-reviewer`를 별도 서브에이전트로 두고, 도구 권한까지 `Read, Grep, Glob, Bash`로 좁힌다. 작성자가 쓸 수 있는 도구와 리뷰어가 쓸 수 있는 도구를 다르게 두는 것은 권한 설계이자 인지 설계다.

### B. 세 개의 병렬 표면과 하나의 이행 순서

구성은 목적이 겹치는 세 표면으로 나뉜다.

| 표면 | 개수 | 성격 | 발동 |
| :--- | ---: | :--- | :--- |
| `agents/` | 72 | 위임 단위, 도구·모델이 프런트매터에 고정 | 명시 호출 또는 자동 위임 |
| `skills/` | 327 | 워크플로·도메인 지식, 주 표면 | 설명 기반 자동 발동 |
| `commands/` | 98 | 슬래시 진입점, 호환 셤 | 사용자 명시 입력 |

읽어야 할 것은 개수가 아니라 **이행 선언**이다. "새 워크플로 개발은 `skills/`에 먼저 들어가야 하고, `commands/`는 마이그레이션 동안 유지되는 호환 계층"이라고 README가 명시한다. 은퇴한 짧은 이름 셤은 `legacy-command-shims/`로 내려 명시적 옵트인 없이는 보이지 않는다. 표면이 셋인 저장소에서 흔히 생기는 실패는 같은 기능이 세 곳에 조금씩 다르게 존재하는 것이다. ECC는 그 실패를 문서 규칙과 디렉터리 분리로 관리한다.

`agents/`의 구성은 언어별 대칭이 눈에 띈다. `go-reviewer`와 `go-build-resolver`, `rust-reviewer`와 `rust-build-resolver`, `java-reviewer`와 `java-build-resolver`, `kotlin-reviewer`와 `kotlin-build-resolver`처럼 **읽기 역할과 고치기 역할**을 쌍으로 둔다. 리뷰는 판단만 하고 수정 권한을 갖지 않으며, 빌드 복구는 실패 로그에 국소화된다. 이 분리는 에이전트가 리뷰 중에 코드를 고쳐 리뷰 대상을 스스로 바꾸는 흔한 실패를 구조적으로 막는다.

### C. 컨텍스트 예산: 숫자로 적힌 계약

ECC가 다른 대형 스킬 팩과 갈라지는 지점은 예산을 **환경변수와 기본값으로** 적어 둔다는 것이다.

$$
C_{\text{sys}} + C_{\text{rules}} + C_{\text{index}} + C_{\text{instincts}} + C_{\text{history}} + C_{\text{MCP}} \le C_{\max}
$$

각 항에는 실제 상한이 붙는다. `SessionStart`가 추가하는 컨텍스트는 기본 8,000자로 상한이 걸리고, 주입되는 instinct는 기본 6개·최소 신뢰도 0.7이다. MCP에 대해서는 권고가 더 노골적이다. 도구 설명이 컨텍스트를 먹어 200k 창이 **약 70k까지 줄 수 있으므로** 프로젝트당 MCP 10개, 활성 도구 80개 이하로 유지하라고 적는다. 토큰 최적화 표는 기본값과 권장값을 나란히 둔다.

| 설정 | 기본 | 권장 | 명시된 효과 |
| :--- | :--- | :--- | :--- |
| `model` | opus | **sonnet** | 비용 약 60% 절감, 코딩 작업 80% 이상 처리 |
| `CLAUDE_CODE_EFFORT_LEVEL` | 모델 의존 | **medium** | 지원 모델에서 추론 깊이를 지연·비용과 교환 |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` | 95 | **50** | 더 이른 압축으로 장세션 품질 유지 |
| `CLAUDE_CODE_SUBAGENT_MODEL` | — | **haiku** | 서브에이전트 비용 하향 |

이 네 줄은 하네스 성능 담론에서 드문 종류의 진술이다. 대부분의 팩은 "가볍다"고 말하지만 ECC는 어느 파라미터를 어느 방향으로 움직이라고 말한다. 물론 이는 측정이 아니라 권고이며, 4절에서 그 구분을 다시 다룬다.

### D. Instincts: 확률적 발동을 결정적 관측으로 바꾸기

기억 층의 설계는 한 문장으로 요약된다. v1은 스킬로 관측했고, 스킬은 모델 판단에 따라 50–80%만 발화하므로 관측이 불완전했다. v2는 `PreToolUse`·`PostToolUse` 훅으로 옮겨 **모든 도구 호출을** 관측한다.

수집된 관측은 백그라운드 옵저버(Haiku)가 5분 간격으로 읽고, 최소 20개 관측이 쌓이면 패턴 탐지를 돌린다. 산출물은 풀 스킬이 아니라 **원자 명제**다. 하나의 트리거, 하나의 행동, 하나의 신뢰도, 근거 목록을 갖는다.

```yaml
id: prefer-functional-style
trigger: "when writing new functions"
confidence: 0.7
domain: "code-style"
scope: project
project_id: "a1b2c3d4e5f6"
```

신뢰도는 0.3–0.9 구간에서 움직이고, 저장소는 증가·감소 조건을 문서화한다. 반복 관측·사용자 미교정·다른 출처의 동의는 올리고, 명시적 교정·장기 미관측·반증은 내린다. 계수는 공개되지 않으므로, 공개된 계약을 다음처럼 형식화할 수 있다.

$$
c_{t+1} = \mathrm{clip}\left(c_t + \alpha \mathbb{1}[\text{support}] - \beta \mathbb{1}[\text{correction}] - \gamma \mathbb{1}[\text{stale}], 0.3, 0.9\right)
$$

주입은 임계값과 상위 $k$의 결합이다. 프로젝트 범위와 스택 일치에 작은 가산 $\lambda$를 주고 상위 6개를 고른다.

$$
S_i = c_i + \lambda \mathbb{1}[\text{project or stack match}], \qquad \mathrm{Inject} = \mathrm{Top6}\lbrace i : c_i \ge 0.7 \rbrace
$$

스코프 설계가 이 구조의 핵심이다. v2.1은 저장 위치를 `~/.claude` 밖으로 옮기고, `CLAUDE_PROJECT_DIR`, `git remote get-url origin`, `git rev-parse --show-toplevel` 순으로 프로젝트를 식별해 12자 해시로 저장한다. 리모트 URL을 쓰는 이유가 좋다. 같은 저장소가 다른 기기에서도 **같은 해시**를 얻어 프로젝트 기억이 이식된다. 승격 규칙도 명시적이다. 같은 instinct가 2개 이상 프로젝트에서 관측되고 평균 신뢰도가 0.8 이상이면 전역 후보가 된다.

이 설계의 실질적 기여는 **오염 방지**다. React 프로젝트의 스타일 관습이 Python 프로젝트로 새지 않게 하면서, "입력을 검증하라" 같은 보편 명제는 전역으로 올린다. 기억을 하나의 평면이 아니라 스코프가 있는 계층으로 둔다는 발상은 에이전트 메모리 연구에서 자주 말해지지만, 파일 레이아웃과 CLI 명령 여섯 개(`/instinct-status`, `/evolve`, `/instinct-export`, `/instinct-import`, `/promote`, `/projects`)로 구현된 사례는 드물다.

### E. AgentShield: 에이전트 구성을 감사하는 정적 분석기

구성 표면이 실행 가능하다면 그것은 코드이고, 코드는 감사 대상이다. AgentShield는 그 논리로 만들어진 별도 바이너리다. 102개 정적 규칙, 1,282개 테스트, 98% 커버리지를 내세우고, 다섯 범주를 본다.

| 범주 | 보는 것 |
| :--- | :--- |
| 비밀 탐지 | 14개 패턴으로 키·토큰 노출 검사 |
| 권한 감사 | 설정의 허용 범위가 과도한가 |
| 훅 주입 | 훅 명령에 주입 가능한 구성이 있는가 |
| MCP 위험 | 서버 정의와 권한 프로파일 |
| 에이전트 구성 | 지시문·스킬의 위험 패턴 |

`--opus` 플래그는 세 개의 Opus 4.6 에이전트를 **공격자·방어자·감사자** 파이프라인으로 돌린다. 공격자가 익스플로잇 체인을 찾고, 방어자가 보호를 평가하고, 감사자가 둘을 종합해 우선순위 위험 평가를 낸다. 출력은 터미널(A–F 등급), JSON, Markdown, HTML이고, 치명 발견 시 종료 코드 2로 빌드 게이트를 닫는다.

주목할 것은 저장소가 **감사기의 출처까지 문제 삼는다**는 점이다. "이 명령들은 이미 설치되고 검토된 AgentShield 바이너리를 요구한다. 레지스트리 게시만으로는 감사가 성립하지 않으며, ECC는 감사된 버전 핀을 제공하지 않는다." 보안 도구를 보안 문제로 다루는 태도이고, 동시에 4절에서 지적할 약점이기도 하다.

### F. 운영 표면의 함정: 훅 이중 로딩과 파괴 명령 게이트

실무에서 가장 자주 깨지는 지점은 훅이다. Claude Code 2.1+는 설치된 플러그인의 `hooks/hooks.json`을 관례적으로 **자동 로드**하므로, `plugin.json`에 `hooks` 필드를 선언하면 중복 감지 오류가 난다.

```text
Duplicate hooks file detected: ./hooks/hooks.json resolves to already-loaded file
```

저장소는 이 문제로 반복된 수정·되돌림 이력을 이슈 번호와 함께 공개하고, 재도입을 막는 회귀 테스트를 둔다. 하네스 생태계의 실제 비용이 여기에 있다. 호스트 동작이 버전 사이에서 바뀌면 사용자 설정이 조용히 두 번 실행되고, 증상은 "왜 느리고 왜 두 번 커밋되지"로 나타난다.

파괴 명령 게이트(`GateGuard`)는 `rm`, 강제·경로 지정 `git checkout`, 파괴적 `find -exec` 같은 명령을 실행 전에 막는다. 공급망 IOC 스캐너는 CI에서 돈다. 세 층(차단, 관측, 감사)이 서로 다른 시점에 놓인 구성이다.

---

## 3. 구조적 비교 (Comparative Positioning)

| 항목 | 단일 시스템 프롬프트 | 스킬 팩(도메인) | 방법론 팩 | ECC |
| :--- | :--- | :--- | :--- | :--- |
| 설치 단위 | 문단 | 스킬 문서 | 스킬 + 부트스트랩 | 플러그인·에이전트·스킬·훅·규칙 |
| 절차 강제 | 없음 | 게이트 산문 | 부트스트랩 주입 + 필수 호출 | 루프 단계 + 훅 이벤트 |
| 기억 | 대화 로그 | 세션 내 | 계획·원장 파일 | 스코프 instinct + 신뢰도 |
| 관측 방식 | 없음 | 모델 판단 | SessionStart 주입 | PreToolUse·PostToolUse 훅 |
| 구성 감사 | 없음 | 없음 | 없음 | AgentShield 102 규칙 |
| 컨텍스트 규율 | 없음 | 색인/본문 분리 | 20,000자 상한 테스트 | 예산 상한을 환경변수로 |
| 하네스 지원 | 호스트 | 호스트별 | 다수(패리티 비대칭) | Claude Code 주 + 어댑터 |

같은 카테고리의 두 문서와 나란히 두면 역할이 갈린다. `obra/superpowers`는 **방법론**을 부트스트랩으로 주입해 "스킬을 반드시 먼저 호출하라"는 규율을 세우고, `elementalsouls/Claude-BugHunter`는 **도메인 지식**을 색인하고 판정 게이트를 붙인다. ECC는 그 둘을 모두 포함하려는 야심을 갖고, 대신 **런타임 표면**(훅, 메모리, 감사기, 예산)을 전면에 둔다. 그래서 ECC의 문서는 "어떻게 일하는가"보다 "무엇이 설치되는가"를 먼저 말한다. 2,215줄의 README와 1,532개 문서 파일이 그 야심의 부산물이다.

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **능력 향상에 대한 측정이 없다**: 72개 에이전트와 327개 스킬이 실제로 산출 품질을 올린다는 벤치마크는 저장소에 없다. 토큰 최적화 표의 "약 60% 절감"은 비용 진술이고, "80% 이상의 코딩 작업을 처리"는 모델 선택에 대한 서술이다. 절차 준수율·결함 밀도·리뷰 적중률 같은 지표는 공개되지 않는다.
2. **표면의 삼중화가 곧 중복 위험**: 에이전트 72, 스킬 327, 커맨드 98은 서로 다른 입구로 같은 일을 한다. 마이그레이션 중이라는 설명은 정당하지만, 사용자 입장에서는 "어느 것을 불러야 하는가"가 매번 문제다. 327개 스킬의 설명이 컨텍스트에 상주하는 비용과, 유사 스킬 간 오발동 가능성은 측정되지 않았다.
3. **신뢰도는 자기 채점이다**: instinct의 0.3–0.9는 모델이 관측에서 부여한 값이지 외부 검증된 빈도가 아니다. 신뢰도가 높은 잘못된 관습이 프로젝트를 오염시키는 경로가 열려 있고, 저장소에는 정밀도 측정도, 부정 관측에 대한 통계도 없다.
4. **훅 결합의 취약성**: 훅 자동 로딩은 호스트 버전에 의존하고, 실제로 이 저장소는 그 변화로 반복 수정을 겪었다. 옵저버는 WSL2·Linux·macOS에서만 동작하고, 네이티브 Windows에서는 프로세스가 훅 종료와 함께 죽어 "성공을 보고하지만 분석은 돌지 않는" 무음 실패가 된다.
5. **감사기의 신뢰 사슬이 열려 있다**: AgentShield는 별도 npm 패키지이고, 저장소는 감사된 버전 핀을 제공하지 않는다고 스스로 밝힌다. 보안 도구를 도입하는 행위 자체가 공급망 위험이 된다는 정직한 고백이지만, 동시에 사용자에게 검증 부담을 넘기는 구조다.
6. **플랫폼 패리티가 비대칭**: Claude Code가 1차 대상이고 Codex·Qoder가 네이티브 경로이며, Cursor·OpenCode·Gemini·Zed·Copilot·Antigravity·Qwen은 "능력이 제한된 어댑터"다. 같은 이름의 스킬이 하네스에 따라 다른 것을 한다는 뜻이고, README도 지원 매트릭스를 먼저 보라고 경고한다.
7. **문서 질량이 유지비**: 2,215줄 README, 1,532개 문서 파일, 17개 언어 번역본이 동시에 갱신되어야 한다. 번역과 본문의 드리프트는 이미 수치 불일치(스킬 327개 대 문서 내 혼용)로 나타난다. 지식 코퍼스에서 문서량은 자산이자 부채다.
8. **예산 규칙은 강제되지 않는다**: MCP 10개·도구 80개·autocompact 50%는 모두 사용자가 설정해야 하는 값이다. 위반을 막는 코드는 없고, 초과 시의 실패는 조용한 품질 저하로 나타난다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **절차를 확률에서 결정으로 옮기는 설계**: "스킬은 50–80% 발화하고 훅은 100% 발화한다"는 관측은 에이전트 시스템 전반에 일반화된다. 어떤 규율을 프롬프트에 둘지, 이벤트에 묶을지 판단하는 기준은 그 규율의 실패 비용과 발화 확률의 곱이다.
- **스코프가 있는 기억**: 프로젝트 해시로 기억을 분리하고 2개 이상 프로젝트에서 반복될 때만 전역으로 승격하는 규칙은, 연속 학습에서 오염과 전이 사이의 절충을 다루는 최소한의 완결된 형태다. 승격 임계값과 최소 신뢰도의 캘리브레이션이 자연스러운 후속 질문이다.
- **자기 구성을 감사 대상으로**: 훅·MCP·권한·비밀을 정적 분석하고, 공격자·방어자·감사자 삼단 파이프라인으로 위험을 종합하는 구성은 에이전트 보안의 한 장르를 연다. 남은 문제는 감사기 자체의 무결성과, 규칙 102개의 오탐률이다.
- **예산을 계약으로 적기**: 컨텍스트 상한을 환경변수 기본값으로 두는 방식은 측정 가능한 형태의 규율이다. 같은 방식으로 스킬 인덱스 비용, MCP 도구 수, 주입 기억 수를 통제하면 팩 간 비교가 가능해진다.
- **후속 연구 방향**: (i) 절차 준수율과 산출 품질의 인과 측정, (ii) 327개 스킬 라우팅의 정밀도·재현율 평가, (iii) instinct 신뢰도의 캘리브레이션과 반증 처리, (iv) 훅 기반 관측의 프라이버시 비용(모든 도구 호출이 기록된다), (v) 하네스 간 어댑터 패리티를 정량화하는 표준 지표.
