# 🤖 CLI Agent Orchestrator: Multi-Agent Supervision over tmux PTYs — Screen-Derived State, Bracketed-Paste Inboxes, and Submit Confirmation

- **Repository**: [awslabs/cli-agent-orchestrator](https://github.com/awslabs/cli-agent-orchestrator)
- **Stars**: ⭐ 1,681 (forks 352, created 2025-07-29, last commit 2026-11-26)
- **Category**: `Agent` / Multi-Agent Orchestration Runtime for Interactive Coding CLIs
- **Project**: CLI Agent Orchestrator (CAO); PyPI `cli-agent-orchestrator` 2.5.5; documentation and two interactive courses at awslabs.github.io/cli-agent-orchestrator; web UI served at `http://localhost:9889`
- **License / Runtime**: Apache-2.0 / Python 3.11+, tmux 3.3+, `uv`; local client-server (FastAPI), SQLite state, React web UI, optional OpenTelemetry; 13 provider CLIs supported
- **Key Themes**: unmodified provider CLIs kept as full processes with their native authentication, tmux window as the unit of an agent, terminal state inferred from the rendered screen (pyte composite with quiescence debounce), inbox delivery by bracketed paste with a submit-confirmation witness, deny-wins handling of human dialogs, role-based tool attenuation with a delegation ceiling, snapshot-before-teardown for traceability

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

멀티 에이전트 시스템을 만들 때 가장 먼저 부딪히는 것은 모델이 아니라 **실행 표면**이다. 상용 코딩 CLI(Claude Code, Codex, Kiro, Copilot CLI 등)는 라이브러리가 아니라 대화형 프로그램이다. API가 없고, 상태를 묻는 엔드포인트가 없고, 인증은 각자의 방식으로 이미 끝나 있다. 이 저장소는 그 사실을 결함이 아니라 **전제**로 삼는다. 에이전트를 재구현하지 않고, 원래의 CLI 프로세스를 그대로 띄워 둔 채로 감독자(supervisor)가 작업을 위임한다.

그 선택은 다섯 개의 병목을 만든다.

1. **상태 관측.** 프로세스가 살아 있는지, 턴이 끝났는지, 사람의 답을 기다리는지 알려주는 채널이 없다. 남은 것은 **PTY에 흘러나온 바이트**와 그것이 그린 화면뿐이다.
2. **메시지 전달.** 다른 에이전트에게 일을 넘기려면 그 에이전트의 TUI 입력창에 텍스트를 넣고 제출해야 한다. 문제는 TUI마다 입력창이 다르고, 처리 중에는 입력을 버리거나 큐에 넣고, 때로는 대화상자가 열려 있어 입력이 **선택지 답변으로 소비**된다는 점이다.
3. **이질성.** 13개의 서로 다른 CLI가 서로 다른 초기화 화면, 프롬프트 표식, 종료 표식을 갖는다. 공통 계약 없이 N×M 조합이 생긴다.
4. **생명주기와 자원.** 터미널은 tmux 창이고, 창은 쌓인다. 끝난 자식을 언제 지우고, 그 전에 무엇을 남길 것인가.
5. **정책.** 감독자는 파일을 읽기만 해야 하고, 리뷰어는 쓰면 안 되며, 자식에게 넘길 수 있는 권한에도 상한이 있어야 한다. 그런데 강제 수단은 각 CLI의 자체 권한 시스템뿐이다.

이 다섯 병목에 대한 이 시스템의 답은 "가능한 한 얇게, 가능한 한 원본 그대로"다. 대신 각 층의 실패 모드를 문서로 명시하고, 실패를 **관측 가능한 상태 전이**로 바꾼다.

---

## 2. 아키텍처 및 수학적 정식화 (Architecture & Formulation)

### A. 클라이언트-서버와 네 개의 제어면

런타임은 로컬 클라이언트-서버 애플리케이션이고, 진입점이 넷이다.

| 표면 | 구현 | 용도 |
| :--- | :--- | :--- |
| `cao` | Click CLI (`cli/`) | 설치·실행·세션·스케줄·설정 |
| `cao-server` | FastAPI (`api/main.py`) | HTTP, SSE, AG-UI, PTY WebSocket |
| `cao-mcp-server` | MCP (`mcp_server/`) | 세션 **안**의 감독자·작업자 도구 35개 |
| `cao-ops-mcp-server` | MCP (`ops_mcp_server/`) | 외부에서 함대를 관리하는 운영 도구 |
| 웹 UI / MCP Apps | React (`web/`, `cao_mcp_apps/`) | 브라우저·호스트 렌더링 인터페이스 |

요청 경로는 한 방향으로 고정된다.

```
CLI, Web UI, MCP client
  -> FastAPI route
  -> session/terminal service
  -> terminal backend
  -> provider adapter
  -> provider CLI process
```

여기서 **터미널 백엔드**와 **프로바이더 어댑터**의 책임 분리가 핵심 규약이다. 백엔드는 터미널 생명주기와 PTY 상호작용을 소유하고(tmux 구현, herdr 구현), 어댑터는 프로바이더별 실행 명령·초기화·상태 감지·입력 처리·종료 동작을 소유한다. 새 프로바이더를 추가할 때 백엔드를 가정하지 않도록 이 경계가 코드 리뷰 규칙으로 명시되어 있다.

### B. 단위: 세션, 터미널, 그리고 스크롤백

세션은 tmux 세션이고 이름은 항상 `cao-` 접두사를 갖는다. 터미널은 tmux 창 하나와 SQLite 레코드 하나다. 이 단순한 동일성이 여러 성질을 만든다.

- 사용자는 `tmux attach`로 감독자와 작업자의 화면을 **그대로** 볼 수 있다. 추적은 별도 로그가 아니라 실제 실행 화면이다.
- 응답은 창의 스크롤백에서 읽어 온다. 그래서 스크롤백 깊이가 정확성 문제가 된다. tmux 기본값 2,000줄은 이 시스템이 시도하는 최심 읽기 5,000줄보다 얕아서, 긴 응답과 TUI의 턴 종료 재그리기가 겹치면 응답이 통째로 밀려나 감독자는 잘린 답이나 빈 답을 받았다. 구현은 팬을 만들기 전에 tmux 서버의 **전역** `history-limit`을 50,000으로 올린다(사용자가 더 높게 잡아 두었으면 내리지 않는다). 50,000줄 × 100자 기준 tmux 메모리는 약 38MB이고 2,000줄 기준은 약 1.8MB다. 메모리는 출력이 도착할 때 게으르게 할당되므로 실제로 채우는 팬만 비용을 낸다.
- 삭제 전에 스냅샷을 남긴다. `$CAO_HOME_DIR/logs/terminal/`에 `<id>.scrollback`(전체 팬 텍스트)과 `<id>.snapshot.json`(세션·창·프로필·프로바이더·작업 디렉터리·허용 도구·호출자)을 0600으로 쓰고, 14일 뒤 정리한다. 스냅샷은 **베스트 에포트**이며, 세션 종료 경로에서는 팬을 죽이기 **전에** 명시적으로 캡처한다(스크롤백은 팬이 살아 있을 때만 존재하므로).

### C. 상태를 화면에서 읽는다: 두 에지와 중간 프로브

터미널 상태는 `IDLE`, `PROCESSING`, `COMPLETED`, `WAITING_USER_ANSWER`, `ERROR`의 다섯 값이다. 이 값을 얻는 방법이 이 시스템에서 가장 조심스러운 부분이다.

원시 바이트 스트림을 그대로 패턴 매칭하면 TUI의 재그리기, 커서 이동, 부분 프레임이 섞여 상태가 요동친다. 구현은 `pyte` 터미널 에뮬레이터로 각 터미널의 출력을 **합성 화면**으로 렌더링하고(400×200), 그 화면 위에서 판정한다. 다만 렌더링만으로는 부족하다. 디바운스 없는 렌더링 판정은 중간 재그리기 프레임을 잡아 원시 경로보다 **더 나쁘다는 것이 측정으로 확인**됐다. 그래서 판정은 두 에지에서만 돈다.

- **상승 에지** — 조용하던 터미널에 출력이 재개되면 `PROCESSING` 가능성이 높다.
- **정착(quiescence)** — 새 출력이 0.2초 동안 없으면 재그리기가 멈춘 것으로 보고 `COMPLETED`/`IDLE`/`WAITING`을 판정한다.

여기에 한 가지 예외가 붙는다. 1초마다 스피너를 다시 그리는 TUI는 결코 정착하지 않으므로, 에지 전용 판정은 상승 에지의 프레임(대개 이전의 준비 상태)만 보고 그 턴 내내 `IDLE`로 읽힌다(Codex 0.153에서 실제 관측). 그래서 버스트가 진행 중이고 아직 `PROCESSING`을 관측하지 못한 동안에는 최대 1초에 한 번 화면을 **중간 프로브**한다. 단, 이 반쪽짜리 프레임에서는 `PROCESSING` 판정만 적용하고 준비 상태 판정은 여전히 정착을 기다린다. 비대칭 적용이 요점이다. 거짓 `PROCESSING`은 전달을 지연시킬 뿐이지만, 거짓 `IDLE`은 처리 중인 에이전트에게 메시지를 밀어 넣는다.

화면 판정은 프로바이더가 `supports_screen_detection`으로 옵트인한 경우에만 쓰이고(현재 Claude Code, Kimi CLI), 나머지는 원시 스트림 경로를 쓴다. 기본값은 켜짐이고 `CAO_PYTE_STATUS=false`로 끈다.

### D. 인박스: 괄호 붙여넣기와 네 개의 전달 경로

`send_message(receiver_id, message)`는 메시지를 DB에 적재하고(PENDING), 대상 터미널의 입력 영역에 **bracketed paste**로 붙여넣은 뒤 Enter를 보낸다. 여기에 tmux 버전별 함정이 있다.

- **tmux 3.7 미만** — CAO가 `ESC[200~` / `ESC[201~` 마커를 직접 감싸고 `paste-buffer -r`로 붙여넣는다. bracketed paste 모드(DECSET 2004)를 스스로 켜지 않는 TUI에서도 여러 줄 메시지가 한 번의 입력으로 도착하게 하려는 것이다.
- **tmux 3.7 이상** — 붙여넣기 내용이 `vis(3)` 살균을 거치므로 손으로 만든 마커는 `^[[200~`이라는 글자 쓰레기로 렌더링된다. 이 버전은 원시 바이트만 적재하고 `paste-buffer -p`로 붙여넣어, tmux가 팬의 DECSET 2004 상태에 따라 진짜 마커를 조건부로 내보내게 한다. 2004를 켜지 않는 TUI는 원시 텍스트를 받아 줄마다 제출된다. 이것은 tmux가 승인한 의미론이고, 살균을 우회하는 `-S`는 쓰지 않는다.

전달은 네 경로가 같은 게이트(`deliver_pending`)로 수렴한다.

| 경로 | 트리거 | 역할 |
| :--- | :--- | :--- |
| 즉시 | `POST /terminals/{id}/inbox/messages` | 적재 직후 1회 시도(5초 대기, `wait_for_outcome` 시 45초) |
| 상태 이벤트 | `terminal.*.status` (IDLE/COMPLETED) | 턴이 끝날 때마다 시도 |
| OpenCode 폴러 | 5초 주기 | 출력이 멈춘 TUI를 위한 빠른 기상 |
| 화해 스윕 | 30초 주기, 30초 유예 | 위 셋이 놓친 PENDING 회수 |

중복 전달은 두 겹의 잠금으로 막는다. 서버 내부에서는 터미널별 잠금이 읽기–표시–전송 시퀀스 전체를 감싸고, 서버 사이에서는 `$CAO_HOME_DIR/server.lock`에 대한 배타 `flock`으로 한 데이터 디렉터리에 서버 하나만 뜨게 한다. 후자가 없으면 두 서버가 같은 PENDING 집합을 각자 스윕해 중복을 만든다.

게이트는 기본적으로 `IDLE`/`COMPLETED`에서만 열린다. 그런데 여기에 두 개의 예외가 있고, 둘 다 **사람의 대화상자**와 관련된다.

1. **대화상자 보호.** 터미널이 `WAITING_USER_ANSWER`일 때, 프로바이더가 `blocks_orchestrated_input_while_waiting_user_answer`를 세우면 오케스트레이션 입력을 전부 거부한다. 선택 위젯이 붙여넣은 텍스트를 **답변으로 소비**하고 뒤따르는 Enter가 하이라이트된 항목을 제출하면, 인박스 메시지가 사람에게 예약된 질문에 답하게 되기 때문이다. 거부된 메시지는 PENDING으로 되돌아가 대화상자가 닫힌 뒤 전달된다.
2. **열성 전달(eager).** Claude Code처럼 처리 중에도 입력을 버퍼링하는 TUI에는 턴 사이 지연을 없애기 위해 `PROCESSING` 중에도 전달한다. 이 경로는 **두 개의 스위치**가 모두 켜져야 한다. 환경변수 `CAO_EAGER_INBOX_DELIVERY=true`(운영자 킬 스위치)와 프로바이더 속성 `accepts_input_while_processing`(프로바이더별 옵트인)이다. 기본값은 둘 다 꺼짐이다.

### E. 제출 확인: 삼켜진 Enter를 어떻게 아는가

붙여넣기–지연–Enter는 **추측**이다. TUI가 준비되지 않았을 때 Enter를 삼키면 메시지는 입력창에 남고 터미널은 여전히 `IDLE`로 읽힌다. 예외도 없고, 메시지는 이미 DELIVERED이므로 발신자는 영원히 기다리고, 다음 메시지가 그 위에 덧붙는다.

그래서 전달은 "붙여넣었다"가 아니라 "**수신자가 이 전달을 처리하기 시작했다**"를 증거로 요구한다. 증거는 상태 전이와 출력 세대 카운터의 결합이다. 터미널마다 출력 청크가 도착할 때마다 증가하는 단조 카운터 $G$가 있고, 키가 팬에 닿기 직전 `dispatch_input` 안에서 그 값을 표본으로 잡아 **전달 경계** $b$로 둔다. 확인 조건은 다음과 같다.

$$
\text{confirm}(b) = \exists \sigma \text{ such that } t_\sigma > t_b \text{ and } G(t_\sigma) > b \text{ and } \sigma \in \lbrace \text{PROCESSING}, \text{COMPLETED}, \text{WAITING} \rbrace
$$

즉 이전 턴에서 래치된 상태는 증거가 되지 못한다. 상태가 바뀌었더라도 **이 전달 이후에 새 출력이 있었다**는 사실이 함께 필요하다.

증거가 8초 안에 오지 않으면 복구가 최대 2회 돈다. 복구는 먼저 프로바이더의 **입력창(composer)을 읽을 수 있는지**를 본다.

| 프로바이더가 할 수 있는 것 | 복구 |
| :--- | :--- |
| 입력창을 읽고 텍스트가 그대로 있음 (codex, claude_code, kiro_cli) | Enter만 다시 |
| 입력창을 읽었는데 텍스트가 사라졌고 직접 상태 프로브가 있음 | 전체 재전송 |
| 입력창을 읽을 수 없음 (나머지 11종) | 아무것도 보내지 않고 유실 기록 |

입력창을 읽을 수 없는데 전체 재전송을 하면 두 번째 사본이 첫 번째 위에 붙어 **둘 다 제출**된다. 그래서 그 경로는 존재하지 않는다. 현재 입력창 판독기와 직접 프로브를 동시에 가진 프로바이더는 없어서, "전체 재전송" 행은 어떤 출하 프로바이더에도 도달하지 않는다. 그리고 어떤 복구도 열린 대화상자에 답하지 않는다. Enter만 다시 보내는 경로는 대화상자 가드를 스스로 검사한다. 붙여넣지 않고 제출하는 유일한 복구가 하이라이트된 선택지를 사람 대신 고르는 경로가 되면 안 되기 때문이다.

비용은 정직하게 계산되어 있다. 정상 전달은 상태 읽기 1–2회를 낸다. 유실된 전달은 **그 터미널의** 잠금을 첫 대기와 실제로 보낸 복구당 한 번씩, 즉 최대 다음만큼 잡는다.

$$
T_{\text{hold}} \le w(1 + k_{\max}) = 8 \times 3 = 24 \text{ s}
$$

다른 터미널은 영향받지 않는다. 전달마다 별도 워커 스레드에서 돌고 스윕은 팬아웃(8)하기 때문이다.

### F. 위임의 의미론

감독자는 두 가지 방식으로 자식을 만든다.

| 도구 | 성질 | 정리 |
| :--- | :--- | :--- |
| `assign` | 비차단. 콜백 보장(`expect_callback` 기본 참) — 자식이 조용히 유휴가 되면 마지막 응답을 감독자에게 전달하고, 자식이 끝나면 종료를 알린다 | 수동 `delete_terminal` |
| `handoff` | 차단. 결과를 기다리고 자식을 자동 삭제(스냅샷은 남긴다) | 자동 |

`assign`에는 격리 수단이 붙는다. `use_worktree=true`는 작업자에게 자기 git 워크트리를 주어 "같은 브랜치를 동시에 편집하는 경쟁"을 없앤다(체크아웃 내용은 삭제 시 버려지고, 병합되지 않은 커밋이 있으면 브랜치는 남는다). `target_host`는 원격 CAO 노드에 작업자를 배치하고 결과는 자동 라우팅한다. `assign_elastic`은 브로커를 통해 탄력 작업자 풀에 위임한다. 또 형제 발견(`list_siblings`)과 메타데이터 갱신으로 작업자끼리 직접 조율하는 경로가 있다.

### G. 권한 감쇠와 위임 상한

도구 정책은 두 층이다. 높은 층의 `role`이 낮은 층의 `allowedTools`로 사상되고, 명시적 `allowedTools`가 항상 역할을 덮어쓴다.

| 역할 | 기본 허용 | 위임 상한 |
| :--- | :--- | :--- |
| `supervisor` | CAO MCP 도구, `fs_read`, `fs_list` | 모든 도구(`*`) |
| `developer` | 내장, `fs_*`, `execute_bash`, `web_fetch`, CAO MCP | 자기 목록 |
| `reviewer` | 내장, `fs_read`, `fs_list`, CAO MCP | 자기 목록(읽기 전용) |
| `workflow_scout` | 내장, `fs_read`, `execute_bash`, CAO MCP | 자기 목록 |

상한의 의미는 다음과 같은 포함 관계다. 부모가 자식에게 부여할 수 있는 권한 집합은 부모의 위임 상한을 넘지 못한다.

$$
A_{\text{child}} \subseteq D_{\text{parent}}
$$

`supervisor`만 기본 상한이 전체이고, 나머지 역할은 자기 목록이 곧 상한이다. 흥미로운 정직함이 문서에 있다. `workflow_scout`은 `fs_write`와 `web_fetch`를 빼앗지만 `execute_bash`는 남기는데, 그 셸은 **완전한 셸**이라 파일 쓰기와 네트워크 접근이 가능하다. 즉 그것은 샌드박스가 아니라 관례다. `@builtin` 선택자는 어떤 프로바이더에서도 그 자체로는 아무것도 부여하지 않는다는 점도 명시된다. 이름이 읽기 좋게 남아 있을 뿐이다.

프로바이더 강제는 각 CLI의 자체 권한 시스템으로 내려간다(Claude Code의 `--allowedTools`, Codex의 설정 등). 오케스트레이터가 강제하는 층과 프로바이더가 강제하는 층이 다르므로, 도구 이름 사상이 프로바이더별로 검증되어야 한다.

### H. 환경 변수 전달의 거부 목록

`cao launch --env KEY=VALUE`는 감독자와 이후 그 세션에 태어나는 모든 작업자에게 변수를 전달한다. 기본 전달은 좁은 허용 목록(`HOME`, `PATH`, `SHELL`, 그리고 `CAO_*`/`KIRO_*`/`MISE_*`/`AWS_*` 접두사)이고, 추가분에는 **거부 목록**이 적용된다. 로더 계열(`LD_*`, `DYLD_*`, `GCONV_PATH`), 셸 시작 훅(`BASH_ENV`, `ZDOTDIR`, `PROMPT_COMMAND`, `PS0`–`PS4`), 인터프리터 훅(`PYTHONSTARTUP`, `PYTHONPATH`, `NODE_OPTIONS`, `RUBYOPT`, `PERL5OPT`), 그리고 `AWS_CONFIG_FILE`처럼 자격증명 파일 위치를 바꾸는 변수가 거부된다. 그 값들은 에이전트의 첫 도구 호출 **이전에** 운영자 권한으로 실행되기 때문이다. 값은 2,048바이트 미만으로 제한되고, 요청 본문으로 전달되어 서버 접근 로그에 남지 않는다. 거부 목록은 새 프로바이더와 새 시작 훅을 뒤따를 수밖에 없다는 한계도 함께 적혀 있다.

### I. 파이프라인과 스케줄

`cao schedule`(구 `cao flow`)은 cron 표현식으로 에이전트 세션을 예약한다. 흐름 파일은 YAML 프런트매터와 프롬프트 본문이고, `script:`가 있으면 그 스크립트가 먼저 돌아 `[[var]]` 자리를 채운 뒤 조건부로 실행된다. 정해진 시각에 조사하고, 조건이 맞을 때만 에이전트를 띄우는 형태가 기본 예제다. `cao-server`가 떠 있어야 실행된다는 점이 이 구조의 제약이다.

---

## 3. 정량 벤치마크 (Quantitative Evidence)

이 저장소는 모델 성능이나 과제 성공률 벤치마크를 공개하지 않는다. 정량 근거는 **프로토콜 상수, 능력 행렬, 자원 회계**다. 확인되지 않은 수치는 넣지 않는다.

### A. 표면 규모

| 항목 | 값 |
| :--- | :--- |
| 세션 내 MCP 도구 | 35개(선언 38개, 기능 플래그로 중복 선언 포함) |
| 지원 프로바이더 CLI | 13종 (Kiro, Claude Code, Codex, Antigravity, Hermes, Kimi, Qwen, MiniMax, Copilot, OpenCode, OMP, Cursor, Grok Build) |
| 패키지 버전 | 2.5.5 |
| 문서 | `docs/*.md` 약 18,900줄, 코드베이스 가이드 별도 |
| 최소 환경 | Python 3.11+, tmux 3.3+, uv, 인증된 프로바이더 CLI 1개 이상 |

### B. 전달 경로의 시간 상수

| 상수 | 값 | 의미 |
| :--- | :--- | :--- |
| `INBOX_POLLING_INTERVAL` | 5초 | OpenCode 폴러 주기 |
| `INBOX_RECONCILE_INTERVAL` | 30초 | 화해 스윕 주기 |
| `INBOX_RECONCILE_GRACE_SECONDS` | 30초 | 스윕이 개입하지 않는 유예 |
| `INBOX_SWEEP_FANOUT` | 8 | 스윕 워커 스레드 |
| `INBOX_SUBMIT_CONFIRM_TIMEOUT` | 8초 | 제출 증거 대기 |
| `INBOX_SUBMIT_MAX_RESUBMITS` | 2 | 복구 횟수 상한 |
| 최대 잠금 점유 | 24초 | 8초 × (1 + 2) |
| `INBOX_IMMEDIATE_DELIVERY_WAIT` | 5초 | POST 응답 경계 |
| `INBOX_DELIVERY_OUTCOME_WAIT` | 45초 | 결과 대기 옵션 시 |

### C. 화면 판정의 상수와 근거

| 상수 | 값 | 근거 |
| :--- | :--- | :--- |
| 합성 화면 크기 | 400×200 | 붙은 터미널이 더 크면 tmux가 팬을 늘려 입력창이 합성 화면 아래로 내려가므로 여유 있게 |
| 정착 디바운스 | 0.2초 | 디바운스 없는 렌더 판정은 원시 경로보다 나쁨(측정) |
| 중간 프로브 | 1.0초 | 1초 스피너 TUI가 정착하지 않는 문제 |
| 스크롤백 상한 | 50,000줄 | 최심 읽기 5,000줄 + TUI 재그리기 여유 |
| 스크롤백 메모리 | 약 38MB / 2,000줄 약 1.8MB | 100자 50,000줄 기준, 도착 시 게으른 할당 |

### D. 프로바이더별 복구 능력

이 행렬이 실패 모드의 실체다. 13종 중 입력창 판독기는 3종(codex, claude_code, kiro_cli), 직접 상태 프로브는 3종(kimi_cli, opencode_cli, mcode)에 있고, **11종은 둘 다 없다**(겹침 포함). 즉 대부분의 프로바이더에서 Enter가 삼켜지면 탐지되지 않고, 그 사실이 로그에 남을 뿐이다.

| 능력 | 프로바이더 |
| :--- | :--- |
| 입력창 판독 | codex, claude_code, kiro_cli |
| 직접 상태 프로브 | kimi_cli, opencode_cli, mcode |
| 처리 중 입력 버퍼링(열성 전달 가능) | claude_code, minimax_code |
| 대화상자가 입력을 소비(가드 대상) | claude_code, codex, minimax_code |

### E. 자원 회계

| 항목 | 값 |
| :--- | :--- |
| 세션 내 터미널 알림 | 10개 도달 시 `assign`/`handoff` 응답에 정리 권고 |
| 끝난 자식 보관(기본 꺼짐) | 마지막 활동 후 60분, 세션당 최대 5개, 초과 시 오래된 것 정리 |
| 스냅샷 권한·보존 | 0600, 14일 |
| 스냅샷 보장 | 모든 삭제 경로에서 시도, 실패는 기록 후 계속(흐름 재활용과 크래시는 캡처 없음) |

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **화면 스크래핑이 근본 상한이다.** 조용한 TUI, 스피너, 커서 이동, 부분 재그리기는 모두 상태 판정의 잡음이다. 프로바이더마다 표식이 다르고, 판정기는 그 표식에 맞춰 조정된다. 프로바이더가 화면을 바꾸면 오케스트레이션의 정확성이 함께 흔들린다.
2. **값비싼 버그가 이미 있었다.** 처리 중 입력을 받아들이는 TUI에서는 Enter가 삼켜져 메시지가 입력창에 남고, 수신자는 영원히 기다린다. 이 유형은 상태가 `IDLE`로 보이기 때문에 관측으로는 정상처럼 보인다. 제출 확인은 그 구멍을 막지만, 확인이 꺼지는 세 경우(설정으로 끔, 열성 전달, herdr TUI 백엔드)에는 여전히 열려 있다. herdr 백엔드는 asyncio 이벤트 루프를 막지 않으려고 확인을 통째로 건너뛴다.
3. **대화상자 충돌은 부류로 존재한다.** 오케스트레이터가 사람에게 열린 질문에 답할 수 있다는 것은 권한 문제이지 편의 문제가 아니다. 방어는 프로바이더별 불리언 플래그에 의존하고, 플래그가 틀리면 메시지는 DELIVERED로 표시되지만 실제로는 사라지고 선택지가 제출된다. 기본값을 보수적으로 두는 것 외에 형식적 보장은 없다.
4. **붙여넣기 의미론이 tmux 버전에 종속된다.** 3.7 미만에서 bracketed paste를 스스로 켜지 않는 TUI는 여러 줄 메시지를 줄마다 제출한다. 이것은 우회하지 않기로 한 결정이며, 다중 행 지시문이 잘게 쪼개져 실행되는 결과를 낳는다.
5. **프로바이더별 복구 커버리지가 얇다.** 11종에 입력창 판독기가 없어, 삼켜진 Enter는 로그만 남는다. 이슈로 추적되고 있지만 현재 상태는 비대칭이다.
6. **단일 서버·단일 데이터 디렉터리.** `flock`이 두 번째 서버를 거부하므로 수평 확장은 원격 노드(`target_host`, Kubernetes 예제)로만 가능하다. 로컬에서는 한 프로세스가 모든 세션의 상태 쓰기를 조정한다.
7. **터미널은 쌓이고, 정리는 사용자의 몫이다.** `assign` 터미널은 자동 삭제되지 않고, 10개에서 권고가 나온다. 끝난 자식을 남기면 추적은 좋아지지만 메모리와 창이 늘어난다.
8. **복원은 에이전트가 아니라 셸이다.** `cao terminal restore`는 원래 세션에 일반 셸 창을 만들고 스크롤백을 재생할 뿐, 프로바이더에 다시 연결하지 않는다. 세션이 종료됐으면 복원은 실패하고 파일만 읽을 수 있다.
9. **정책 표현력의 구멍.** `execute_bash`는 완전한 셸이므로 `workflow_scout`의 읽기 전용 의도는 관례다. `@builtin`은 아무것도 부여하지 않는데 이름 때문에 부여한다고 오해하기 쉽다. `--yolo`는 모든 제한을 우회한다. 역할 이름이 내장 역할과 겹치면 설정의 목록이 내장을 **완전히 대체**하며, `["*"]`까지 넓힐 수 있다.
10. **환경 변수 거부 목록은 뒤처진다.** 새 프로바이더의 인증 훅이나 새 셸 시작 파일이 추가되면 목록을 갱신해야 하고, 그 사이에는 우회 경로가 생긴다. 저자들도 이를 명시한다.
11. **에이전트는 사용자의 자격증명으로 산다.** 이것이 "네이티브 인증 유지"의 대가다. 작업자 프로세스는 사용자의 CLI 세션과 같은 권한을 갖고, 작업자 출력은 텍스트로 감독자의 컨텍스트에 다시 들어간다. 즉 작업자 출력은 **외부 텍스트**이며, 프롬프트 주입의 표면이다.
12. **플랫폼 의존.** tmux 3.3+가 필수이므로 사실상 POSIX 계열이다. Windows는 WSL이나 원격 노드를 거쳐야 한다. 웹 UI 자산은 배포본에 포함되지만 개발에는 Node 툴체인이 필요하다.
13. **확장 기능이 기본으로 닫혀 있다.** 에이전트 플러그인 표면(CLI, TUI, 웹 패널, HTTP 라우트)은 이름 결정 전까지 게이트되어 있고, 환경변수 없이는 404를 돌려준다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **PTY를 통합 기반으로 쓰는 전략.** API 없는 도구를 오케스트레이션하려면 재구현(충실도 상실)과 래핑(취약함) 사이에서 골라야 한다. 이 시스템은 래핑을 택하고, 취약함을 **명시적 상태 기계와 증거 기반 확인**으로 흡수한다. 사내 도구·레거시 GUI·상용 CLI를 묶는 실무에서 그대로 참조할 수 있는 구조다.
- **제출 확인이라는 일반 패턴.** "붙여넣고 Enter"는 텍스트를 목적지에 넣는 모든 시스템(터미널, 채팅 입력창, 폼)에서 같은 실패를 낳는다. 출력 세대 카운터라는 **단조 증인**과 전달 경계 표본으로 "이 전달이 처리되기 시작했다"를 판정하는 방식은, 블라인드 쓰기 API의 신뢰성을 올리는 값싼 장치다.
- **거짓 상태의 비대칭 비용.** 중간 프로브에서 `PROCESSING`만 적용하고 준비 상태는 정착까지 기다리는 규칙은, 오류 비용이 대칭이 아니라는 사실을 코드로 옮긴 예다. 거짓 양성은 지연, 거짓 음성은 데이터 손상이다.
- **거부 우선(deny-wins) 정책.** 대화상자 앞에서 오케스트레이션 입력을 전부 막는 것은, 권한 시스템에서 흔히 쓰는 규칙을 에이전트 상호작용에 적용한 것이다. 새 기능(열성 전달)이 이 규칙을 우회하지 못하도록 두 스위치 게이트를 둔 점도 함께 볼 만하다.
- **권한 감쇠와 위임 상한.** `allowed`와 `delegate`를 분리하고 상한을 역할에 묶는 설계는 객체 능력(capability) 모델의 축소판이다. 다만 셸이 있으면 그 격자는 무너진다는 사실을 숨기지 않는다. 에이전트 권한 설계에서 "무엇을 줄 수 있는가"와 "무엇을 넘길 수 있는가"를 분리하는 표준적 출발점이 된다.
- **teardown 전 스냅샷.** 삭제 경로마다 스크롤백을 먼저 캡처하는 규율은, 실행 흔적이 사라지는 순간을 정확히 아는 시스템의 설계다. 워크숍·디버깅·감사에서 "무엇을 했는가"를 화면 그대로 남기는 값싼 방법이다.
- **자원 회계의 투명성.** 스크롤백 50,000줄이 약 38MB라는 식으로 메모리 비용을 문서에 적는 관행은, 기본값을 바꾸는 결정의 근거를 사용자와 공유한다.
- **후속 연구 방향**: (i) 입력창 판독기를 프로바이더 전반으로 확장(11종 공백), (ii) 구조화된 위임 프로토콜(요약이 아니라 검증 가능한 산출물 계약), (iii) PTY 세션의 결정론적 재현·리플레이, (iv) 다중 노드에서의 세션 상태 일관성, (v) 작업자 간 통신의 신뢰 경계(프롬프트 주입 격리), (vi) 화면 판정을 프로바이더 네이티브 이벤트(구조화 로그·JSON 출력)와 결합했을 때의 정확도·비용 곡선.

### 참고 링크

- 저장소: https://github.com/awslabs/cli-agent-orchestrator
- 문서: https://awslabs.github.io/cli-agent-orchestrator/
- 코드베이스 지도: https://github.com/awslabs/cli-agent-orchestrator/blob/main/CODEBASE.md
- 인박스 전달(붙여넣기·확인·복구): https://github.com/awslabs/cli-agent-orchestrator/blob/main/docs/inbox-delivery.md
- 터미널 생명주기·스냅샷: https://github.com/awslabs/cli-agent-orchestrator/blob/main/docs/terminal-lifecycle.md
- tmux 세션·스크롤백·환경 전달: https://github.com/awslabs/cli-agent-orchestrator/blob/main/docs/tmux.md
- 도구 제한·역할·위임 상한: https://github.com/awslabs/cli-agent-orchestrator/blob/main/docs/tool-restrictions.md
- 에이전트 프로필 스키마: https://github.com/awslabs/cli-agent-orchestrator/blob/main/docs/agent-profile.md
- 예약 실행(흐름): https://github.com/awslabs/cli-agent-orchestrator/blob/main/docs/flows.md
- 화면 판정 상수: https://github.com/awslabs/cli-agent-orchestrator/blob/main/src/cli_agent_orchestrator/constants.py
