# ⚡ agent-browser: A Shell-Native Browser Runtime for Coding Agents — Daemon-Mediated CDP Sessions, Document-Scoped Durable Refs, and Containment as a Per-Command Contract

- **Repository**: [vercel-labs/agent-browser](https://github.com/vercel-labs/agent-browser)
- **Stars**: ⭐ 51,600+ (forks 3,600+, created 2026-01, 587 open issues)
- **Category**: `Harness` / Browser Execution Runtime & Per-Command Containment
- **Organization**: Vercel Labs (Apache-2.0)
- **License / Runtime**: Apache-2.0 / Rust CLI + Rust daemon over the Chrome DevTools Protocol (about 144k LOC under `cli/src`, 2,263 in-source test attributes, 14 integration test files); npm `agent-browser` v0.43.0 with 120 changelog releases, Homebrew, and native binaries for macOS–Linux–Windows; Node.js 22+ only for the launcher, browser work is native
- **Key Themes**: one-command-per-step shell loop over a per-session daemon, accessibility snapshots with `@e1`-style refs keyed by `(page, frame, loaderId, backendNodeId)`, 160 MCP tools quantized into 8 startup profiles (core = 33), AES-256-GCM auth vault that never enters model context, opt-in domain allowlist with WebRTC suppression, action policy with fail-closed loading, per-process CSPRNG content boundaries, bounded WebMCP proactive discovery (4 KiB / 16 tools / 250 ms), recorded-flow replay over first-party HTTP, skill and eval corpus tested by ablation

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

코딩 에이전트가 브라우저를 쓰는 방식은 대부분 "셸에서 명령 한 줄"이다. 에이전트의 도구 호출은 프로세스 생성이고, 결과는 표준출력 텍스트이며, 다음 판단은 그 텍스트를 읽은 모델이 내린다. 이 인터페이스는 이식성이 좋지만 세 가지 구조적 병목을 만든다.

1. **관측 비용(Observation Cost)**: 페이지 하나를 모델이 읽을 수 있는 형태로 바꾸는 비용이다. DOM 직렬화는 문서 크기에 비례하고, 스크린샷은 텍스트를 잃으며 좌표만 남긴다. 접근성 트리는 그 중간이지만 노드 수가 곧 토큰이므로, 상호작용 대상만 남기는 프루닝이 필수다.
2. **참조 안정성(Referential Stability)**: 셸 명령 사이에는 프로세스가 사라진다. 셀렉터는 리팩터링에, 좌표는 레이아웃 이동에 취약하고, "관측한 대상"과 "행동한 대상"이 같은 노드라는 보장이 없다. 참조가 프로세스 수명보다 오래 살아남으려면 정체성이 프로세스 밖에 있어야 한다.
3. **경계(Containment)**: 에이전트가 읽는 페이지는 신뢰할 수 없는 입력이다. 페이지 안의 문장이 도구 출력처럼 보일 수 있고, 자격증명이 프롬프트에 들어가면 유출 경로가 열리며, 조작된 에이전트가 임의 도메인으로 데이터를 보낼 수 있다. 이 위협들은 "모델이 지시를 따르지 않기를 바라는" 방식으로는 막을 수 없다.

프로젝트의 설계 응답은 세 가지로 요약된다. 상태를 프로세스가 아니라 **세션 데몬**에 두고, 참조를 셀렉터가 아니라 **문서 스코프의 노드 정체성**에 두며, 경계를 기본값이 아니라 **명령마다 실리는 계약**으로 둔다. 마지막 선택은 논쟁적이다. 저장소는 "탐색·행동·출력 제어는 옵트인이며 기본적으로 아무 제한도 걸지 않는다"고 명시한다. 안전이 기본값이 아니라 설정이라는 뜻이고, 이 문서의 4절은 그 대가를 따로 계산한다.

---

## 2. 아키텍처 및 수학적 정식화 (Architecture & Formulation)

### A. 데몬 매개 세션: 상태 수명과 프로세스 수명의 분리

CLI는 얇은 클라이언트다. 첫 명령이 세션별 데몬을 띄우고, 이후 명령은 소켓으로 그 데몬에 붙어 같은 브라우저를 조작한다. `--session`마다 데몬과 브라우저가 하나씩 생기고, 기본값은 헤드리스이며, 명령이 1시간(`DEFAULT_IDLE_TIMEOUT_MS = 3,600,000`) 없으면 세션 지속성이 설정된 경우 상태를 저장한 뒤 스스로 종료한다. 상태 없는 실행과 데몬 실행의 비용 차이는 명령 수 $k$에 대해 다음과 같다.

$$
C_{\text{stateless}}(k) = k\,(c_{\text{spawn}} + c_{\text{launch}} + c_{\text{step}}), \qquad C_{\text{daemon}}(k) = c_{\text{spawn}} + c_{\text{launch}} + k\,(c_{\text{spawn}} + c_{\text{ipc}} + c_{\text{step}})
$$

브라우저 기동 $c_{\text{launch}}$는 명령당 한 번이 아니라 세션당 한 번으로 상각된다. 대신 데몬과 브라우저의 상주 메모리가 최대 1시간 유지되고, 세션 상태가 프로세스 경계를 넘으므로 **설정 변경**이라는 새로운 문제가 생긴다.

설정은 명령마다 실려 오고, 데몬은 실행 중 값과의 차이만 적용한다. 생략하면 실행 값 유지, 같은 값이면 무동작(같은 `--profile`을 매번 넘겨도 재기동하지 않는다), 타임아웃·정책·확인·다이얼로그·복원 설정은 제자리 적용, 실행 설정(`--profile`, `--args`, `--proxy`, `--state`, `--allowed-domains` 등)은 브라우저만 재기동한다. 적용된 변경은 표준오류에 한 줄씩, JSON 응답의 `lifecycle.settings.changes`에 `applied` 플래그와 함께 남는다. 비밀은 양쪽에서 마스킹된다.

보안 설정은 별도 취급을 받는다. `--allowed-domains`, `--action-policy`, `--confirm-actions`는 소켓 옆의 소유자 전용 파일(`<session>.settings`)에 **새 값이 효력이 생기기 전에** 기록되어, 유휴 만료·크래시·강제 종료·업그레이드를 넘겨 살아남는다. 파일이 읽히지 않거나 파싱되지 않거나 다른 사용자가 읽을 수 있으면 데몬은 시작을 거부하고 파일 이름을 알린다. **격리 설정이 조용히 사라지는 대신, 기동이 시끄럽게 실패한다**는 선택이다. 실패한 기동에서도 `--allowed-domains`만은 설치된 채 유지되어, 경계가 넓어지는 방향으로는 절대 되돌아가지 않는다.

### B. 관측 표현과 참조 정체성

`snapshot`은 접근성 트리를 직렬화한다. `-i`(상호작용 요소), `-c`(빈 구조 제거), `-d`(깊이), `-s`(셀렉터 스코프)로 출력을 깎고, 이름이 비었거나 같은 스냅샷 안에서 중복된 컨트롤에는 참조 뒤에 힌트를 붙인다(`icon:lucide-trash`, `img:cart`, `data-testid:`, `name:`, `id:`, `in:Blue mug`). 힌트는 접근성 이름이 아니므로 `find role --name` 매칭에는 쓰이지 않고, 행동은 참조로만 한다는 원칙을 흔들지 않는다.

참조의 정체성은 네 튜플로 결정된다. 페이지 세션 $p$, 프레임 $f$, 문서 로더 $\lambda$, 백엔드 노드 번호 $\beta$에 대해,

$$
\mathrm{ref}(n) = D\bigl[(p,\, f,\, \lambda,\, \beta(n))\bigr]
$$

$D$는 문서별로 유지되는 사상이고, 스냅샷마다 비워지는 것은 일시 사상이다. 그래서 살아남은 DOM 요소는 같은 참조를 유지하고, 요소가 교체되거나 문서가 바뀌면($\lambda$ 변경) 참조는 재사용 없이 무효화된다. 로더를 모르는 문서는 참조를 이어받지 않는다. 저장소의 회귀 테스트가 이 세 성질을 그대로 고정한다.

무효화된 참조는 조용히 다른 요소를 가리키지 않는다. 남아 있는 정체성으로 복구를 시도하되, 같은 역할·이름의 노드가 정확히 하나일 때만 받아들인다.

$$
\mathrm{recover}(r, m) \;\text{성공} \iff \bigl\lvert \lbrace n : \mathrm{role}(n) = r \wedge \mathrm{name}(n) = m \rbrace \bigr\rvert = 1
$$

둘이면 "여러 요소가 일치하니 새 스냅샷을 찍으라"는 오류로 끝난다. 가용성을 내주고 오작동을 막는, 이 저장소에서 반복되는 실패 정책이다.

### C. 경계의 세 층: 도메인, 행동, 내용

**도메인 허용목록**은 탐색뿐 아니라 하위 리소스, WebSocket, EventSource, `sendBeacon`까지 막고, 지원되는 Chromium 세션에서는 WebRTC 피어 연결을 꺼서 STUN·TURN·관련 DNS로 HTTP 가로채기를 우회하지 못하게 한다. 명령 하나가 진행되는 조건은 다음과 같다.

$$
\mathrm{proceed}(a, u) \iff \mathrm{host}(u) \in A \;\wedge\; \pi(a) \ne \mathrm{deny} \;\wedge\; \bigl(\pi(a) = \mathrm{allow} \;\vee\; \mathrm{confirmed}(a, 60\,\mathrm{s})\bigr)
$$

**행동 정책**은 `allow`·`deny`·`confirm` 목록과 기본값으로 구성된 JSON이다. 정책 파일을 읽을 수 없으면 `close`를 제외한 모든 행동을 거부하는 정책이 만들어진다(코드에 명시된 fail-closed). 확인 대기는 60초 뒤 자동 거부되고, `--confirm-interactive`가 켜졌는데 표준입력이 터미널이 아니면 즉시 거부된다. 파이프 입력에서의 사고를 구조적으로 막는 조치다.

**내용 경계**는 페이지에서 온 텍스트를 도구 출력과 구분하는 표식이다. 본문 앞뒤를 `AGENT_BROWSER_PAGE_CONTENT`와 `END_AGENT_BROWSER_PAGE_CONTENT`로 감싸고, 프로세스마다 CSPRNG로 만든 nonce $\nu$를 양쪽 표식에 함께 넣는다.

$$
\mathrm{marker} = \bigl(\mathrm{label},\, \nu,\, \mathrm{origin}\bigr), \qquad \nu \sim \mathrm{CSPRNG}(\cdot)
$$

문서 스스로 "방어 심층"이라 밝히듯, 이 층은 오케스트레이터가 표식을 존중한다는 가정 위에 선다. nonce는 위조를 비현실적으로 만들 뿐, 모델이 표식 안의 지시를 따르지 않게 강제하지는 않는다.

**자격증명**은 세 번째 축이다. `auth save`로 저장한 프로필은 `~/.agent-browser/auth/`에 AES-256-GCM으로 암호화되고, 키가 없으면 `~/.agent-browser/.encryption-key`에 자동 생성되며, 파일 권한은 Unix `600/700`, Windows `icacls`로 제한된다. 로그인 시 자격증명은 데몬의 IPC 채널을 지나지 않고 CLI가 직접 다루며, 모델 컨텍스트에는 이름과 URL만 노출된다. `--no-navigate` 모드는 기존 문서에서 폼을 채우되, 채우기 전과 자격증명마다 스킴·호스트·유효 포트가 일치하는지 다시 검사한다. 플러그인은 `agent-browser.plugin.v1` stdio 프로토콜로 프로세스 밖에서 돌고 `credential.read`·`browser.provider`·`launch.mutate`·`command.run` 같은 능력 단위로만 접근한다.

### D. 도구 표면의 이중화와 프로필 양자화

같은 레지스트리가 CLI와 MCP 두 전송으로 노출된다. MCP 도구는 160개이고, 기동 프로필은 8개(core 33, network 9, state 27, debug 40, tabs 13, react 8, mobile 15, webmcp 4)와 `all`로 나뉜다. 기본값은 core다.

$$
\frac{\lvert T_{\text{core}} \rvert}{\lvert T_{\text{all}} \rvert} = \frac{33}{160} \approx 0.206
$$

요청마다 고정으로 지불되는 스키마 비용 $\sum_{t \in T} s(t)$를 약 80% 줄이는 대신, 페이지 진단·네트워크 검사·프레임 제어 같은 능력을 포기한다. 프로필은 쉼표로 합성할 수 있지만(`--tools core,network,react`), 과업 난이도에 따라 턴마다 도구 집합을 고르는 적응형 노출은 없다.

패리티는 선언이 아니라 구조로 강제된다. MCP 호출은 같은 바이너리를 `--json` 모드로 다시 호출하는 방식으로 CLI 파서에 위임하고, 저장소의 `AGENTS.md`는 CLI 명령·플래그·동작·환경변수를 바꿀 때 `cli/src/mcp.rs`를 같은 변경에서 갱신하라고 요구하며, 도구 카탈로그 스냅샷 테스트가 그 표면을 고정한다. 전송이 둘이어도 의미가 하나라는 성질을 테스트로 유지하는 흔치 않은 사례다.

### E. 관측의 대안: HTTP 읽기, 스냅샷 차분, 픽셀 차분

브라우저를 띄우는 것이 항상 옳지는 않다. `read`는 브라우저를 거치지 않고 HTTP로 본문을 가져오며 `text/markdown`을 우선 협상하고 `llms.txt`/`llms-full.txt`를 인식한다. 정적 문서에는 이것이 스냅샷보다 훨씬 싸다. `diff`는 두 방향을 제공한다. 스냅샷 차분은 텍스트 줄 단위 추가·삭제·불변 수를 세고, 스크린샷 차분은 정규화된 RGB 거리로 임계값을 적용한다.

$$
\delta(p) = \frac{\lVert c_A(p) - c_B(p) \rVert_2}{\sqrt{3} \cdot 255}, \qquad \rho = \frac{\bigl\lvert \lbrace p : \delta(p) \gt t \rbrace \bigr\rvert}{W H}
$$

임계값 $t$는 설정값이고, 크기가 다르면 비교 없이 불일치 100%로 처리한다. 두 차분은 서로 다른 질문에 답한다. 스냅샷 차분은 "의미가 바뀌었는가", 픽셀 차분은 "보이는 것이 바뀌었는가"다.

### F. 녹화된 흐름의 재생: 행동 궤적의 메모이제이션

`flow`는 브라우저에서 한 번 수행한 과업을 1차 API 요청 시퀀스로 저장하고, 다음부터는 HTTP로 직접 재생한다. 텔레메트리 경로(`/collect`, `/beacon`, `/pixel` 등)는 선택에서 제외되고, 요청 본문은 슬롯이 있는 템플릿으로 일반화된다. 재생 결과가 검증에 실패하면 흐름은 "미검증"으로 표시되고 UI 경로로 되돌아간다.

$$
C_{\text{replay}} = O(1) \text{ HTTP 왕복}, \qquad C_{\text{agent}} = O(k)\;(\text{스냅샷} + \text{행동}) \text{ 왕복}
$$

스냅샷·클릭 루프를 요청 하나로 접는 대신, 사이트 변경에 대한 취약성을 떠안는다. 검증 게이트가 그 취약성을 "조용한 오답"이 아니라 "명시적 실패"로 바꾸는 장치다.

### G. 증거 장치: 스킬 평가와 데몬 벤치마크

`evals/`는 스킬 표면을 다섯 범주(skill-loading, skill-selection, command-usage, reference-routing, context-footprint)로 나눠 `claude`/`codex` CLI를 상대로 채점하고, 선택적으로 1–5점 LLM 심사를 붙인다. `context-footprint.ts`는 얇은 스킬, `skills get core` 출력, MCP `initialize`, core 프로필, `--tools all` 프로필의 바이트와 근사 토큰을 결정론적으로 측정한다. `webmcp-context.py`는 지역 상점 과업을 실제 Codex 에이전트에 던지고, `ordinary`·`hostile`·`hostile-schema`·`hostile-result` 모드로 도구 설명·스키마·결과에 주입된 악성 지시와 가짜 `readOnlyHint`를 구분해 채점하며, 카나리 유출과 무관한 도구 호출을 거부 조건으로 둔다. 저장소는 이를 "신뢰성 주장이 아니라 스모크 평가"라고 스스로 한정한다.

`benchmarks/`는 Vercel Sandbox 마이크로VM에서 명령 지연(내비게이션·스냅샷·스크린샷·평가·클릭·채우기·에이전트 루프·7명령 워크플로), 콜드 스타트, 바이너리·배포 크기, 데몬 RSS·피크 RSS, 브라우저 RSS, CPU 시간, 프로세스 수를 측정한다. 문서가 미리 밝히는 결론은 정직하다. 명령 지연은 CDP 왕복이 지배하므로 데몬 간 차이는 대개 작고, 표준편차 열로 잡음을 걸러야 한다.

---

## 3. 구조적 비교 (Comparative Positioning)

| 항목 | 원시 CDP 스크립트 | Playwright/Puppeteer 라이브러리 | 스크린샷 컴퓨터 유즈 | Chrome DevTools MCP | agent-browser |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 에이전트 인터페이스 | 없음(직접 구현) | 코드 작성 | 이미지 + 좌표 | MCP 도구 63개 | 셸 명령 + MCP 160개 |
| 호출 간 상태 | 세션 코드가 유지 | 프로세스가 유지 | 세션 유지 | MCP 서버가 유지 | 세션 데몬이 유지 |
| 참조 정체성 | 사용자 정의 | 셀렉터 | 좌표 | `(loaderId, backendNodeId)` | `(page, frame, loaderId, backendNodeId)` |
| 기본 격리 | 없음 | 없음 | 없음 | 없음 | 없음(옵트인), 정책 로드는 fail-closed |
| 자격증명 처리 | 사용자 코드 | 사용자 코드 | 프롬프트 노출 위험 | 없음 | 암호화 볼트, 컨텍스트 비노출 |
| 성능 통찰 | 원시 이벤트 | 직접 분석 | 없음 | TraceEngine + Lighthouse | trace·profiler·vitals·react |
| 재현 자동화 | 스크립트 | 스크립트 | 없음 | 없음 | 스냅샷 재생 + HTTP 흐름 재생 |
| 평가 자산 | 없음 | 없음 | 없음 | 없음 | 스킬 평가 5범주 + 주입 스모크 + 데몬 벤치 |

가장 직접적인 비교 대상은 같은 문제를 MCP 단일 전송으로 푸는 Chrome DevTools MCP다. 그쪽은 도구 63개와 접근성 스냅샷, TraceEngine 임베딩으로 "관측의 질"을 밀어붙이고, agent-browser는 셸 루프·데몬·세션·격리·재생으로 "실행의 형태"를 밀어붙인다. 도구 수 160 대 63은 능력의 차이라기보다 표면 철학의 차이다. 전자는 CLI 전체를 MCP로 거울처럼 비추고, 후자는 MCP를 1차 인터페이스로 설계한다.

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **안전이 기본값이 아니다**: 저장소가 명시하듯 탐색·행동·출력 제어는 옵트인이고, 기본값에는 아무 제한이 없다. 격리를 원하면 `--allowed-domains`·`--action-policy`·`--content-boundaries`를 호출자가 매번 챙겨야 한다. fail-closed는 "설정을 켠 뒤"의 성질이지 설치 직후의 성질이 아니다.
2. **허용목록은 OS 방화벽이 아니다**: WebSocket·EventSource 차단은 생성자 재정의 기반이라 최선 노력이고, `eval` 범주를 허용하면 페이지 스크립트가 원래 생성자를 되살릴 수 있다. 클라우드 제공자에서는 연결 전 로딩이 소급 차단되지 않고, `--cdp`·`--auto-connect`·`--profile`·`--state`·iOS·Safari·direct-page 플러그인과는 아예 함께 쓸 수 없다. 경계가 필요한 곳에서는 호스트·컨테이너·네트워크 송신 제어를 겹쳐야 한다.
3. **내용 경계는 권고다**: nonce가 위조를 어렵게 만들 뿐, 모델이 표식을 신뢰 경계로 다루는지는 오케스트레이터의 규율에 달려 있다. 저장소도 "LLM과 오케스트레이터가 표식을 존중한다는 가정"이라고 적는다.
4. **참조의 수명은 짧다**: 내비게이션마다 참조가 죽고, 중복 역할·이름은 복구를 거부한다. 긴 루프에서 참조 재사용은 "요소를 찾을 수 없음"으로 나타나 원인 진단이 어렵고, 복구의 안전성은 가용성과 맞바꾼 것이다.
5. **도구 표면의 비대**: 160개 도구는 요청마다 스키마 비용을 지불하고, 8개 프리셋은 거친 양자화다. 중간 해상도(예: core + debug만)는 조합으로만 가능하고, 과업에 따른 동적 노출은 없다.
6. **자체 과업 성공률 벤치마크의 부재**: 스킬 평가는 "스킬을 제대로 불러오는가"를 재고, WebMCP 스모크는 과업 하나를 3회 반복한다. "브라우저 과업 성공률이 다른 표면보다 높은가"는 이 저장소의 증거로 답할 수 없다. 데몬 벤치마크도 지연과 메모리를 재지 성공률을 재지 않는다.
7. **볼트 키의 동거**: `AGENT_BROWSER_ENCRYPTION_KEY`를 설정하지 않으면 키가 볼트 옆 파일에 자동 생성된다. 파일 권한으로 막지만, 호스트가 뚫리면 키와 볼트가 함께 넘어간다. Windows는 소켓 디렉터리 ACL에 의존한다.
8. **버전 확인의 외부 요청**: Chrome이 뒤처졌는지 판단하려고 Chrome Releases 피드에 접속한다(3초 타임아웃, `AGENT_BROWSER_NO_VERSION_CHECK=1`로 해제). 편의와 프라이버시의 교환이다.
9. **릴리스 속도와 의미 변화**: 10개월 남짓에 120개 릴리스다. 설정 병합 규칙, 프로필, 흐름 재생의 의미가 빠르게 바뀌므로, 버전을 고정하지 않은 자동화는 조용히 다른 동작을 만날 수 있다.
10. **재생의 취약성**: 흐름 재생은 사이트의 요청 형태가 바뀌면 검증 실패로 끝난다. 이는 정직한 실패지만, 실패 시 UI 폴백 비용은 다시 $O(k)$ 루프다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **관측·행동 인터페이스가 성공률의 1차 지렛대**: 접근성 트리 + 문서 스코프 핸들 + 힌트 보강은 "관측 표현이 행동 성공률을 결정한다"는 명제의 비교 가능한 구현이다. 같은 페이지 픽스처에서 셀렉터·좌표·핸들 세 방식을 붙여 성공률과 재시도 수를 재는 실험이 자연스럽다.
- **핸들 정체성을 문서 정체성에 묶는 설계**: `loaderId`를 정체성의 일부로 넣는 선택은, 같은 DOM 노드가 다른 문서에서는 다른 객체라는 사실을 프로토콜 수준에서 반영한다. iframe 교체와 세션 전환을 구분하는 이 규칙은 브라우저 밖(예: 재렌더링되는 UI 트리, 재시작되는 샌드박스)에도 이식된다.
- **fail-closed 설정 로딩**: 정책 파일을 읽지 못하면 전부 거부하고, 권한이 느슨한 설정 파일이면 데몬을 띄우지 않으며, 실패한 기동에서도 격리를 넓히지 않는다. "설정이 사라진 채 조용히 실행"이라는 흔한 사고를 구조적으로 제거하는 패턴으로, 장기 실행 에이전트 런타임의 기본 계약으로 참조할 만하다.
- **능력 양자화의 비용 곡선**: 33/160 비율과 프로필 합성은 "스키마 고정 비용 대 과업 성공률" 곡선을 실험할 수 있는 드문 표면이다. 턴마다 도구 집합을 고르는 적응형 노출은 미해결 연구 주제이며, `context-footprint` 평가가 그 측정 도구의 출발점이 된다.
- **행동 궤적의 메모이제이션과 검증**: 흐름 재생은 에이전트 루프를 API 호출로 접는 캐시이고, 검증 실패를 명시적 상태로 만든다. "언제 캐시를 신뢰하고 언제 다시 관측할 것인가"는 테스트 타임 계산과 신뢰 캘리브레이션이 만나는 문제다.
- **주입 저항의 스모크 벤치마크**: `hostile`·`hostile-schema`·`hostile-result` 모드와 카나리는 페이지 제공 도구(WebMCP)라는 새로운 공격면에 대한 최소 재현 가능한 평가다. 모델·호스트 조합별 주입 성공률을 재는 표준 하네스로 확장할 여지가 크다.
- **후속 연구 방향**: (i) 데몬형·상태없는 도구 표면의 과업 성공률과 총비용 비교, (ii) 참조 복구 정책의 엄격도와 재시도 성공률의 관계, (iii) 격리 기본값이 실제 사고율을 얼마나 줄이는가, (iv) 흐름 재생의 신뢰도 추정과 자동 재검증 주기, (v) 페이지 제공 도구 생태계에서 신뢰 경계를 프롬프트가 아니라 프로토콜로 옮기는 설계.
