# 🤖 AG-UI: A Typed Event Stream Between Agents and Applications — 31 Events in Eight Families, Snapshot-versus-JSON-Patch State, and Interrupt-as-Run-Outcome

- **Repository**: [ag-ui-protocol/ag-ui](https://github.com/ag-ui-protocol/ag-ui)
- **Stars**: ⭐ 19,020 (forks 1,966, created 2025-05, last commit 2026-11-16)
- **Category**: `Agent` / Agent-User Interaction Protocol & Multi-Language SDK
- **Organization**: AG-UI Protocol (created by CopilotKit, developed in the open; adopted by Google, Microsoft, Amazon, Oracle)
- **License / Runtime**: MIT / TypeScript, Python, and .NET SDKs generated from one JSON Schema (`spec/1.0/schema.json`, 98 definitions); community SDKs for Kotlin, Go, Java, Dart, Rust, Ruby, C++, Swift; transport-agnostic (HTTP + SSE and HTTP + Protobuf bindings)
- **Key Themes**: 1.0 frozen specification with a schema-driven code generator, conformance fixtures per transport, run lifecycle as the only mandatory family, state through snapshots plus RFC 6902 deltas, interrupt/resume as a run boundary, subagent attribution on every event, a documented robustness asymmetry between unknown and malformed material

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

에이전트는 요청-응답 모델에 맞지 않는다. 토큰을 흘려보내고, 응답 도중 도구를 호출하고, 공유 상태를 갱신하고, 사람의 승인을 기다리고, 하위 에이전트를 낳는다. 프레임워크마다 이 흐름을 다르게 표현하므로, 앱을 하나 만들 때마다 접착 코드를 새로 쓴다. 이 저장소가 정의하는 병목은 네 가지다.

1. **전송 표면의 분열(Transport Fragmentation)**: 같은 에이전트를 웹, 모바일, 터미널, 슬랙에 붙이려면 네 개의 어댑터가 필요하다. 어댑터마다 부분 업데이트와 오류 처리 의미가 다르다.
2. **상태 동기화의 양방향성(Bidirectional State)**: 에이전트가 앱의 상태를 읽고 쓰며, 앱도 에이전트의 상태를 읽는다. 전체 스냅샷만 주고받으면 대역폭이 터지고, 델타만 보내면 순서가 깨진 소비자가 발산한다.
3. **중단의 표현 부재(Missing Suspension)**: 사람 승인 대기, 구조화 입력 요청, 외부 정책 결정 대기는 흐름 중간의 정지다. 이를 "에러"나 "타임아웃"으로 표현하면 재개 지점을 잃는다.
4. **버전 협상(Version Negotiation)**: 기록된 교환이 자기를 설명하지 못하면 0.x 클라이언트와 1.0 서버가 조용히 어긋난다.

응답은 하나의 **입력 객체 + 타입 있는 이벤트 스트림**이다. 앱은 `RunAgentInput` 하나를 보내고, 에이전트는 8개 계열의 이벤트를 흘려보낸다. 이 단순한 계약이 위 네 병목을 각각 다른 장치로 해결한다: 전송 중립 바인딩, 스냅샷·델타 이중화, 런 결과로서의 중단, 인밴드 프로토콜 버전.

---

## 2. 아키텍처 및 수학적 정식화 (Architecture & Formulation)

### A. 입력 계약의 최소성

`RunAgentInput`은 `threadId`, `runId`, `messages`만 필수다. 스키마 주석이 그 이유를 직접 적는다. "도구와 컨텍스트는 없는 키와 빈 배열이 같은 뜻이므로, 필수로 두면 생산자가 틀릴 수 있는 것을 잡지 못한다." 즉 필수 필드 집합은 **검증력 대비 마찰**의 최적화 결과다. 이 판단은 프로토콜 설계에서 재사용할 만한 규칙이다. 필수로 두는 필드는 "없으면 의미가 달라지는" 것뿐이어야 한다.

`protocolVersion`은 전송 헤더가 아니라 입력 본문에 실린다. 기록된 교환이 그 자체로 자기를 설명하게 만들기 위해서다. 로그를 나중에 읽을 때 어떤 규칙으로 해석해야 하는지가 데이터 안에 있다.

### B. 이벤트 대수

31개 이벤트는 8개 계열로 묶인다.

| 계열 | 이벤트 | 성질 |
| :--- | :--- | :--- |
| 런·스텝 | `RUN_STARTED` `RUN_FINISHED` `RUN_ERROR` `STEP_STARTED` `STEP_FINISHED` | 필수는 런 시작·종료 |
| 텍스트 | `TEXT_MESSAGE_START` `_CONTENT` `_END` `_CHUNK` | 시작-내용-끝 3단, 청크는 편의형 |
| 도구 호출 | `TOOL_CALL_START` `_ARGS` `_END` `_RESULT` `_CHUNK` | 인자 누적 후 결과 |
| 상태 | `STATE_SNAPSHOT` `STATE_DELTA` `MESSAGES_SNAPSHOT` | 절대·상대 이중화 |
| 추론 | `REASONING_START` `REASONING_MESSAGE_*` `REASONING_ENCRYPTED_VALUE` `REASONING_END` | 암호화 블롭 왕복 |
| 활동 | `ACTIVITY_SNAPSHOT` `ACTIVITY_DELTA` | 진행 표시 |
| 서브에이전트 | `SUBAGENT_STARTED` `_FINISHED` `_ERROR` | 귀속 |
| 통과 | `RAW` `CUSTOM` | 임의 확장 |

소비자는 스트림을 왼쪽 접기(fold)로 처리한다. 상태 $s_t$와 이벤트 $e_t$에 대해

$$
s_{t+1} = \delta(s_t, e_t)
$$

이고, 런은 시퀀스 $e_1, \dots, e_n$에 이 접기를 적용한 결과다. 필수 계열이 런 생명주기뿐이라는 점이 확장성을 만든다. 최소 에이전트는 다섯 개 이벤트만 내고, 기능은 계열 단위로 추가된다. 이 구조에서는 "이 계열을 모르는 소비자"가 정상 상태이고, 그래서 C항의 오류 분류가 필요해진다.

### C. 인식 불가와 규격 위반의 비대칭

스펙은 두 경우를 정반대로 다룬다. 인식하지 못하는 이벤트 타입, 설명되지 않은 속성, 나중에 추가된 종류의 결과는 **런을 중단시키면 안 된다**. 반면 프로토콜이 설명하는 필드가 스키마가 거부하는 값을 담으면 **치명적**이고, 소비자는 고치거나 강제 변환하지 말고 런을 실패시켜야 한다. 형식으로 쓰면

$$
\text{type}(e) \notin T \text{ or } \text{prop}(e) \notin \Sigma \quad \implies \quad \text{fatal}(e) = \text{false}
$$

$$
\text{known field with invalid value} \quad \implies \quad \text{fatal}(e) = \text{true}
$$

이 비대칭은 포스텔 원칙의 반쪽 적용이다. 관용은 미래 확장에만 주고, 현재 계약에는 주지 않는다. 조용한 보정이 실제로는 가장 위험한 실패이기 때문이다. 잘못된 `delta`를 "고쳐서" 적용하면 상태는 조용히 오염되고, 그 오염은 이후 모든 판단에 전파된다.

같은 원칙이 청크에도 적용된다. 청크는 그 자체로 하나의 이벤트이고, 확장 이후에 판정되어야 한다. 확장을 먼저 하면 잘못된 청크가 정상처럼 보인다.

### D. 상태 수렴: 스냅샷은 기준점, 델타는 함수

`STATE_SNAPSHOT`은 상태 전체를 대체하고, `STATE_DELTA`는 RFC 6902 패치 목록이다. 패치 $p$를 상태에 적용하는 함수 $\mathrm{apply}$로 보면

$$
s_{t+1} = \mathrm{apply}(s_t, p_t), \qquad s_{t+1} = \mathrm{snap}_t \ \text{(replace)}
$$

두 방식의 차이는 오류 누적이다. 스냅샷은 무상태(idempotent)이므로 몇 번을 잃어도 다음 스냅샷에서 회복한다. 델타는 순서 의존이고 하나를 잃으면 이후 모든 적용이 어긋난다. 그래서 스냅샷은 **재기준화 장치**로 쓰인다. 연결이 끊겼거나 큰 변경이 있을 때 스냅샷을 보내고 델타의 기준을 새로 잡는다.

이 설계는 복제 로그(replicated log)의 표준형이다. 그리고 스키마 주석은 더 정직한 사실을 적는다. "구조적으로 유효한 패치가 적용 가능하다는 뜻은 아니다. 존재하지 않는 경로를 가리키는 연산은 RFC 6902가 적용자에게 맡긴다." 즉 스키마 검증은 1차 필터일 뿐이고, 적용 실패는 별도 경로다. `test` 연산을 쓰면 낙관적 동시성 제어가 가능하지만, 프로토콜은 그 사용을 강제하지 않는다.

### E. 중단과 재개: 런 경계가 커밋 지점

1.0의 중단 모델은 "정지 상태를 유지하는 스트림"이 아니라 "중단 결과로 끝나는 런 + 응답을 실은 새 런"이다.

$$
\text{run}_1 \to \text{RUN-FINISHED}\lbrace \text{outcome} : \text{interrupt}, \ \text{interrupts} \rbrace, \quad
\text{run}_2 = \text{resume}\lbrace \text{interruptId} \mapsto (\text{status}, \text{payload}) \rbrace
$$

`outcome`이 생략되면 성공으로 간주한다. 이 기본값이 0.x 호환의 핵심이다. 중단을 모르던 생산자의 출력이 이미 그 모양이기 때문이다. 재개는 새 `runId`를 받고, `resume` 배열의 `interruptId`가 이전 런의 중단을 가리킨다.

이 구조의 대가는 서버가 재개 지점을 스스로 기억하지 않는다는 점이고, 이득은 그 대가와 정확히 같다. 재개 지점이 클라이언트에 있으므로 서버는 무상태로 남는다. 대신 클라이언트가 잃은 것은 되살릴 수 없다. 스펙은 이를 숨기지 않고 명시한다. SSE 바인딩에는 **스트림 재개가 없다**. `Last-Event-ID`를 쓰지 않고, 끊긴 스트림은 다시 들어갈 수 없다. 다시 실행은 새 `runId`를 가진 새 런이고, 소비자가 보존한 것만 입력으로 넘어간다.

그래서 **잘린 런**은 별도의 1급 결과다. 종료 이벤트 없이 끊긴 연결은 실패가 아니라 "부분"이고, `200` 응답은 성공의 증거가 아니다. 이 구분이 없으면 클라이언트는 잘린 스트림을 완료로 오인한다.

### F. 스키마 단일 소스와 코드 생성

명세는 `schema.json` 하나이고, 생성기(`spec/generator/`)가 중간 표현을 거쳐 TypeScript·Python·.NET 모델을 만든다. Protobuf 번역과 필드 번호 고정(`proto-freeze.txt`)이 별도로 있고, 바이너리 바인딩도 같은 스키마에서 나온다. 검증은 `conformance` 픽스처가 맡는다. 같은 스트림이 JSON으로 살아남으면 protobuf로도 살아남아야 하고, 실패하면 같은 방식으로 실패해야 한다.

이 구성의 성질은 **드리프트 불가능성**이다. SDK가 손으로 쓰이지 않으므로 스키마와 SDK가 어긋날 수 없다. 반대로 스키마 자체의 의미 오류는 세 SDK에 동시에 전파된다. 단일 소스는 일관성을 사고 결함의 반경을 넓힌다.

### G. 상호운용 위치

MCP는 에이전트에 도구와 컨텍스트를 주고, A2A는 에이전트끼리 연결하며, AG-UI는 에이전트를 사용자에게 가져온다. 생성 UI 명세(A2UI, MCP Apps, Open-JSON-UI)는 "무엇을 보여줄지"를 서술하고, AG-UI는 그 페이로드를 포함한 모든 것을 나르는 런타임이다. 이 분업은 프로토콜 설계에서 흔히 혼동되는 두 층(무엇을 / 어떻게)을 분리한다.

---

## 3. 구조적 비교 (Comparative Positioning)

| 항목 | REST + WebSocket 수제 연동 | OpenAI Assistants 스트림 | Vercel AI SDK 데이터 스트림 | MCP | AG-UI |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 타입 있는 이벤트 계약 | 없음(자체 정의) | 부분 | 부분 | 도구 호출 중심 | 31개 이벤트 |
| 공유 상태 동기화 | 직접 구현 | 제한적 | 제한적 | 없음 | 스냅샷 + JSON Patch |
| 사람 승인 중단 | 직접 구현 | 도구 승인 | 제한적 | 없음 | 런 결과 + 재개 |
| 프레임워크 중립 | ✓ | ✗ | 부분 | ✓ | ✓ |
| 다국어 SDK 생성 | ✗ | 일부 | TypeScript 중심 | 다수 | 스키마 → 3개 공식 + 8개 커뮤니티 |
| 적합성 시험 | ✗ | ✗ | ✗ | 있음 | 픽스처 + 전송별 시험 |
| 전송 중립 | ✓ | ✗ | ✗ | ✓ | SSE · Protobuf |

`agent/` 안에서 AG-UI의 자리는 "오케스트레이션(ruflo, CowAgent, Qwen Code) → **에이전트와 사용자 사이의 계약**"이다. 다른 문서들이 에이전트 내부 구조를 다룬다면, 이쪽은 에이전트 바깥 경계를 규정한다. 강점은 명세의 규범적 완결성이고, 약점은 그 규범이 런타임 상태를 얼마나 보존하느냐에 있다(재개 없음).

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **스트림 재개의 부재**: SSE 바인딩은 `Last-Event-ID`를 쓰지 않고 끊긴 스트림을 다시 들어갈 수 없다. 긴 런에서 연결이 끊기면 진행 상황은 버려지고, 소비자가 보존한 상태만으로 새 런을 시작해야 한다. 비용이 큰 도구 호출이 이미 실행된 경우 그 결과를 재현할 방법이 없다.
2. **델타의 순서 의존**: 델타는 적용 순서가 계약의 일부인데, 전송 계층은 순서를 보장하되 유실 시 재전송 수단이 없다. `test` 연산을 쓰지 않으면 어긋남을 감지할 수 없고, 스키마는 그 사용을 권장하지 않는다.
3. **스키마 검증의 한계**: 구조적으로 유효한 패치가 적용 가능하지 않을 수 있다는 점을 스펙 자신이 인정한다. 즉 "검증 통과"는 "상태 일관"을 뜻하지 않는다.
4. **문서와 명세의 불일치**: 개념 문서(`docs/concepts/architecture.mdx`)는 "16개의 표준 이벤트 타입"이라고 적고, 1.0 명세와 README는 31개다. 16개는 0.x 시절의 수이고 갱신되지 않았다. 프로토콜 문서에서 이 종류의 잔재는 통합자에게 직접적 혼선을 만든다.
5. **규범의 양과 진입 장벽**: 처리 규칙 문서는 MUST/SHOULD가 밀도 높게 이어지고, 미들웨어 순서(삭제 전에 관찰, 확장 후에 판정)까지 규정한다. 정확성에는 유리하지만, 최소 구현자가 전부 읽지 않으면 조용히 비준수 소비자가 만들어진다.
6. **추론 암호화 블롭의 불투명성**: `REASONING_ENCRYPTED_VALUE`는 제공자별 암호문을 그대로 왕복시킨다. 앱은 내용을 검증할 수 없고, 제공자 간 이식도 불가능하다. 관측 가능성은 제공자에게 남는다.
7. **서브에이전트 귀속의 신뢰**: 서브에이전트 이벤트는 귀속 정보를 제공하지만, 그것이 실제 실행 구조와 일치하는지는 서버 구현에 달려 있다. 프로토콜은 진위를 검증할 수단을 제공하지 않는다.
8. **전송별 의미 차이의 잔여**: 두 바인딩이 같은 판정을 내려야 한다고 규정하지만, 바이너리 인코딩 자체가 만드는 차이는 예외로 남는다. 그 예외의 범위는 구현자가 해석해야 한다.
9. **버전 협상의 비대칭**: 0.x↔1.0 양방향 호환이라도, 1.0 기능(중단, 서브에이전트)을 0.x 소비자에게 보내면 그 소비자는 이를 모른 채 무시한다. "중단을 모르는 소비자가 중단 런을 성공으로 읽는" 경로는 기본값 규칙 때문에 열려 있다.
10. **생태계 편중**: 통합 목록은 대부분 파이썬·타입스크립트 프레임워크다. 커뮤니티 SDK 여덟 개는 공식 생성기 바깥에서 유지되므로 스키마 변경에 대한 추종 속도가 다르다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **필수 필드의 최소성 원칙**: "없는 것과 빈 것이 같은 뜻이면 필수로 두지 않는다"는 규칙은 모든 스키마 설계에 적용된다. 필수 필드는 검증력이 아니라 의미 분기를 만드는 곳에만 둔다.
- **오류의 비대칭 분류**: 미래 확장에는 관용, 현재 계약 위반에는 치명. 이 비대칭은 장기 유지되는 프로토콜의 표준 패턴으로 삼을 만하다. 조용한 보정은 가장 비싼 실패다.
- **스냅샷·델타 이중화**: 절대 상태와 상대 변경을 함께 두고, 스냅샷을 재기준화 지점으로 쓰는 설계는 실시간 협업, 텔레메트리, 구성 동기화에 그대로 옮길 수 있다. 델타 유실 시 발산을 스냅샷 주기로 제한한다는 성질을 설계 목표로 명시할 수 있다.
- **중단을 결과로 모델링**: 정지 상태를 스트림에 매달지 않고 런 경계로 끊으면 서버가 무상태로 남는다. 대가는 재개 정보의 클라이언트 보관이다. 이 교환을 의식적으로 선택한 사례로 읽을 가치가 있다.
- **스키마 단일 소스와 적합성 픽스처**: 손으로 쓴 SDK를 없애면 드리프트가 사라지고, 대신 스키마 오류의 반경이 커진다. 이 교환을 관리하려면 스키마 리뷰와 픽스처가 SDK 테스트보다 중요해진다.
- **후속 연구 방향**: (i) 델타 유실 하에서의 수렴 보장 조건(스냅샷 주기와 발산 확률의 관계), (ii) 재개 가능한 스트림 바인딩의 설계(웹소켓·재생 로그), (iii) 서브에이전트 귀속의 검증 가능성, (iv) 인밴드 버전 협상의 실패 모드 분류, (v) 규범 문서 준수도를 자동 측정하는 적합성 시험의 확장.
