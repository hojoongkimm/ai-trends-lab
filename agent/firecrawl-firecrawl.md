# 🔥 Firecrawl: Agent Knowledge Infrastructure as a Fallback Waterfall — Engine Priority, Provenance-Labeled Cloud Hand-Off, and a Fail-Safe Local Boundary

- **Repository**: [firecrawl/firecrawl](https://github.com/firecrawl/firecrawl)
- **Stars**: ⭐ 204,261 (forks 10,702, created 2024-04-15, last push 2026-11-20)
- **Category**: `Agent` / Agent Knowledge and Retrieval Infrastructure
- **Organization**: Firecrawl (commercial company behind firecrawl.dev; AGPL-3.0 for the server, MIT for SDKs)
- **License / Runtime**: AGPL-3.0 / TypeScript monorepo for the API and SDKs, Rust for the local binary; Docker Compose with Playwright, Redis, RabbitMQ, NuQ PostgreSQL and optional FoundationDB for self-hosting
- **Key Themes**: one API over scrape, crawl, map, batch scrape, search and 550-plus data providers, a feature-flag-driven engine fallback waterfall with per-engine time budgets, an LLM-ready markdown contract, a local-first binary that serves the same v2 API on localhost with an explicitly labelled cloud hand-off, and a network boundary that refuses metadata endpoints in every mode

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

에이전트가 세상에 대해 아는 것은 결국 수집된 텍스트다. 그 텍스트를 얻는 일은 오래전에 해결된 것처럼 보이지만, 실제로는 네 가지가 동시에 어긋난다.

1. **매체의 이질성(Media Heterogeneity)**: 정적 HTML, 자바스크립트로 그려지는 앱, PDF, 오피스 문서, 이미지, 유튜브가 같은 "페이지"라는 이름 아래 섞여 있다. 어떤 단일 fetcher도 전부를 잘하지 못한다.
2. **차단과 회피(Blocking)**: 사이트는 봇을 막고, 막히면 다른 경로가 필요하다. 경로마다 비용·지연·성공률이 다르다.
3. **계약의 부재(No Contract)**: 수집 결과가 마크다운인지 HTML인지, 어디까지가 원문이고 어디부터가 생성물인지 불분명하면 하류 에이전트는 그 출력을 신뢰할 수 없다.
4. **신뢰 경계(Trust Boundary)**: 수집한 페이지는 곧 프롬프트에 들어간다. 숨은 지시문과 개인정보가 함께 들어오고, 로컬에서 처리한 것과 클라우드로 보낸 것을 구분할 수 없으면 데이터 흐름을 설명할 수 없다.

Firecrawl의 응답은 **엔진 폴백 + 명시적 출처 표기 + 로컬 우선 경계**다. 이 세 가지가 일반 스크래퍼와 이 제품을 가르는 지점이고, 그래서 이 문서는 UI가 아니라 실행 경로를 본다.

---

## 2. 아키텍처 및 정식화 (Architecture & Formulation)

### A. 엔진 집합과 폴백의 구성

스크레이퍼의 본체는 `apps/api/src/scraper/scrapeURL`이고, 여기서 실제 수집을 수행하는 엔진은 다음 열다섯 가지로 선언된다(코드의 `Engine` 타입).

| 계열 | 엔진 | 역할 |
| :--- | :--- | :--- |
| 브라우저 | `fire-engine;chrome-cdp`, `;chrome-cdp;stealth`, 각각의 `(retry)` 변형 | CDP 기반 렌더링, 스텔스 프록시 |
| TLS | `fire-engine;tlsclient`, `;tlsclient;stealth` | 브라우저 없이 TLS 지문을 맞춘 fetch |
| 로컬 브라우저 | `playwright` | 자체 호스팅용 Playwright 마이크로서비스 |
| 정적 | `fetch` | 단순 HTTP |
| 문서 | `pdf`, `document`, `image` | PDF, 오피스, 이미지 OCR |
| 색인 | `index`, `index;documents` | 사전 수집 색인 조회 |
| 외부 | `wikipedia`, `exchange` | 위키피디아 엔터프라이즈, 데이터 제공자 교환 |

선택은 무작위가 아니라 **요구 기능의 최대 우선순위**로 결정된다. 기능 플래그마다 우선순위가 있고(`pdf`·`document`·`image`·`audio`·`video` = 100, `atsv`·`useFastMode` = 90, `actions`·`stealthProxy`·`branding` = 20, `waitFor` = 1), 요청이 요구하는 플래그 집합 $F$에 대해 필요한 등급이 정해진다.

$$
P(F) = \max_{f \in F} \pi(f), \qquad E(F) = \left[ e \in E \mid \pi(e) \ge P(F) \right]
$$

$E(F)$ 안에서 순서는 고정되어 있고(색인 → 파이어엔진 CDP → 스텔스 → 재시도 → TLS → Playwright → fetch → PDF → 문서 → 이미지), 실패하면 다음 엔진으로 내려간다. 각 엔진에는 "합리적 최대 시간"(max reasonable time)이 정의되어 있어, 느린 엔진이 전체 요청을 붙잡지 못한다.

$$
T_{\text{scrape}} \le \min\left(T_{\text{deadline}}, \sum_{e \in \text{attempted}} \min(t_e, m_e)\right)
$$

이 구조의 핵심은 폴백이 **예외 처리가 아니라 정책**이라는 점이다. 실패는 흐름의 일부이고, 어떤 실패가 어떤 다음 엔진을 부르는지가 코드에 이름으로 적혀 있다. 최근 추가된 `waterfallExperiments`는 터널 오류(`ERR_TUNNEL_CONNECTION_FAILED`)에서 재시도 엔진으로 넘어가는 경로를 50% 롤아웃으로 실험하는데, 배정은 `sha256(scrapeId + experiment)`의 앞 32비트를 균등 분포로 사상해 결정한다. 즉 실험군 배정이 무작위가 아니라 **재현 가능한 해시**다.

### B. 변환 파이프라인과 그 위험

엔진이 HTML을 가져오면 변환기(`transformers/`)가 형식을 만든다. 마크다운 변환 외에 JSON 추출(`llmExtract.ts`, 58KB), 질의 응답, 문서 diff, 오디오·비디오 전사, 메뉴·상품 추출, 유튜브 전사, 그리고 두 개의 안전 변환기가 있다. `redactPII`(개인정보 마스킹)와 `promptInjectionGuard`다.

프롬프트 인젝션 가드는 **LLM 기반 형식보다 먼저** 실행된다. 즉 요약·JSON 추출 같은 모델 호출 전에 페이지를 스캔한다. 여기에 정직한 실패 처리가 붙는다. 스캔 가능한 길이 상한 $C_{\max}$가 있고, 페이지가 그보다 길면 전체를 검사하지 못했다는 사실을 결과의 `warning`에 적고 **요금을 부과하지 않는다**.

$$
\text{scanned}(d) \iff \lvert d \rvert \le C_{\max}, \qquad \text{partially scanned}(d) \iff \lvert d \rvert > C_{\max}
$$

검사하지 못한 것을 "깨끗함"으로 반환하지 않는 이 규칙이, 안전 기능에서 가장 흔한 실패 모드(검사 안 함을 통과로 표시)를 막는다.

### C. 두 런타임: 로컬과 클라우드

같은 저장소가 세 가지 배포 형태를 갖는다.

| 항목 | Firecrawl Local | Firecrawl Cloud | 자체 호스팅(`apps/api`) |
| :--- | :--- | :--- | :--- |
| 실행 단위 | Rust 바이너리 하나(crates.io) | 호스팅 서비스 | Docker Compose 스택 |
| 필요 인프라 | 없음(선택: Chrome, Node 22+) | 없음 | Playwright, Redis, RabbitMQ, NuQ PostgreSQL, 선택 FoundationDB |
| scrape/crawl/map | 로컬 | 클라우드 | 자체 서버 |
| search | 클라우드 경유 | 클라우드 | 파이어엔진 설정 시 |
| Alexandria 550+ 제공자 | 로그인 후 클라우드 | 클라우드 | 미포함 |
| v2 API | `localhost:3002` | `api.firecrawl.dev` | 게시한 포트 |
| 작업 상태 | 메모리(시작 후 1시간 만료) | 영속 | 큐 기반 |

로컬 바이너리는 API의 HTML 정제 코드(`apps/api/native/src`)를 **그대로 컴파일**한다. 그래서 로컬 결과와 클라우드 결과의 마크다운이 같은 규칙을 따른다. 크레이트 배포 시에는 그 경로를 볼 수 없으므로 `scripts/vendor-native.sh`가 코드를 복사해 넣고, 저장소 안에서 빌드할 때는 항상 공유 원본을 쓴다. 이중 경로를 만든 이유가 "사본 부패"를 막기 위해서라는 점이 코드에 적혀 있다.

### D. 출처 표기와 열화의 계약

로컬 실행이 실패한 페이지는 클라우드가 대신 가져올 수 있다. 그때 결과에는 `"source": "firecrawl-cloud"`가 붙고, 클라우드도 실패하면 `"blocked": true`와 이유가 담긴 경고가 붙는다. 하류 에이전트는 이 필드로 "내 기계에서 읽은 것"과 "남의 서버가 읽어 준 것"을 구분할 수 있다. 데이터 흐름을 설명해야 하는 조직에게 이 표기는 기능이 아니라 요구사항이다.

클라우드 호출 정책은 3상태이고, **알 수 없는 값은 `off`로 처리**된다.

$$
v \in \left[ \texttt{auto}, \texttt{search} \right] \Rightarrow \text{cloud}(v) = v, \qquad v \notin \left[ \texttt{auto}, \texttt{search} \right] \Rightarrow \text{cloud}(v) = \texttt{off}
$$

기본값이 "가능한 한 로컬"이 아니라 "명시한 것만 클라우드"라는 뜻이며, 설정 오타가 조용히 데이터를 내보내지 않게 만든 선택이다.

### E. 네트워크 경계

로컬 실행의 보안 모델은 실행 모드마다 다르게 설계되어 있다. 사람이 키보드 앞에서 쓰는 `scrape`·`crawl`·`map`·`mcp`는 사설망과 localhost에 접근할 수 있고, `--public-only`로 스스로 제한할 수 있다. 반면 다른 프로그램의 요청을 받는 `serve`는 루프백에만 바인딩하고, 사설 주소는 `--allow-private` 없이는 거부하며, 외부 주소에 바인딩하려면 토큰을 요구하고 허용 Host 목록을 검사한다. 그리고 **모든 모드에서** 링크 로컬과 클라우드 메타데이터 주소는 리다이렉트와 DNS 해석 이후에도 거부된다.

이 규칙이 필요한 이유는 문서에 직접 적혀 있다. 신뢰할 수 없는 페이지를 읽는 에이전트는 그 안의 숨은 지시로 내부 URL을 열게 될 수 있고, 그 한 번의 요청이 클라우드 자격 증명을 내주는 경로가 된다. 그래서 브라우저도 같은 규칙 아래 둔다. Chrome은 헤드리스·샌드박스·일회용 프로필로 띄우고, 그 앞의 루프백 프록시가 리다이렉트·하위 리소스·스크립트 요청·웹소켓까지 검사한다. 컨테이너에서 렌더링하려면 user namespace를 허용하는 seccomp 프로필을 쓰되, `seccomp=unconfined`는 쓰지 말라고 명시한다.

---

## 3. 구조적 비교 (Comparative Positioning)

| 항목 | 단순 fetch + 파서 | 헤드리스 브라우저 단독 | 검색 API | 범용 크롤러 프레임워크 | Firecrawl |
| :--- | :--- | :--- | :--- | :--- | :--- |
| JS 렌더링 | ✗ | ✓ | 부분 | 부분 | ✓ (폴백 포함) |
| PDF·오피스 파싱 | 부분 | ✗ | ✗ | 부분 | ✓ (로컬 포함) |
| 실패 시 대체 경로 | ✗ | ✗ | ✗ | 부분 | ✓ (엔진 폴백) |
| 로컬 전용 실행 | ✓ | ✓ | ✗ | ✓ | ✓ (바이너리 하나) |
| 클라우드/로컬 출처 표기 | ✗ | ✗ | ✗ | ✗ | ✓ (`source` 필드) |
| 프롬프트 인젝션 게이트 | ✗ | ✗ | ✗ | ✗ | ✓ (LLM 형식 이전) |
| 메타데이터 주소 차단 | ✗ | ✗ | ✗ | ✗ | ✓ (전 모드) |
| 큐·워커·확장 | ✗ | ✗ | ✗ | ✓ | ✓ (NuQ, K8s, Helm) |
| LLM 친화 출력 계약 | ✗ | ✗ | 부분 | ✗ | ✓ (마크다운·JSON·스키마) |

`agent/` 안에서의 위치는 "에이전트 두뇌"도 "실행 하네스"도 아닌 **지식 공급 계층**이다. 도구 표면(MCP·CLI·SDK 9개 언어)과 데이터 계약이 에이전트 쪽을 향해 설계되어 있다는 점에서, 같은 카테고리의 에이전트 프레임워크와는 다른 층을 차지한다.

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **AGPL이라는 제약**: 서버가 AGPL-3.0이다. 사내 서비스로 개조해 노출하면 소스 공개 의무가 따라온다. SDK는 MIT라는 점이 완화 요소지만, 서버 임베딩을 검토하는 조직에는 실질적 장벽이다.
2. **검색은 로컬이 아니다**: 색인은 노트북에 없다. `FIRECRAWL_CLOUD=off`로 끄면 검색 기능 자체가 사라진다. "완전 로컬"은 수집에 한정된 주장이다.
3. **벤치마크의 출처**: 웹 커버리지 96%, P95 3.4초는 자사 블로그가 제시한 수치다. 독립 재현이 공개되어 있지 않으므로 마케팅 수치로 분류해야 한다.
4. **로컬 기능의 축소**: 로컬은 `json`·`summary` 형식을 경고와 함께 건너뛰고 브라우저 `actions`를 무시한다. 스크린샷은 Chrome이 필요하다. 크롤·배치 작업은 메모리에 있어 프로세스가 끝나면 사라지고 1시간 뒤 만료된다.
5. **자체 호스팅 기본값의 위험**: Compose 기본값은 인증이 꺼져 있고(`USE_DB_AUTHENTICATION=false`), 게시 포트는 API 하나뿐이며, 큐·캐시·메시지 브로커에 영속 볼륨이 정의되어 있지 않다. 문서가 이를 명시하지만, 기본값을 그대로 운영에 올리는 사고는 반복된다.
6. **인젝션 가드의 범위**: 길이 상한을 넘는 페이지는 부분 검사로 끝난다. 경고와 미과금으로 정직하게 처리하지만, 보호의 범위가 페이지 길이에 반비례한다는 사실은 변하지 않는다.
7. **마크다운의 손실**: 표·주석·구조가 마크다운으로 접히는 과정에서 사라진다. 레이아웃 자체가 정보인 페이지(대시보드, 지도, 상품 비교표)에는 부적합하다.
8. **표면의 과다**: 로컬 CLI, 클라우드 CLI, MCP, 스킬 카탈로그, SDK 9종, 플러그인 마켓플레이스가 동시에 존재한다. 각 표면의 업데이트 주기와 기능 집합이 달라, "무엇이 최신인가"를 사용자가 판단해야 한다.
9. **로컬과 클라우드의 의미 차이**: 같은 `/v2` 경로라도 로컬은 일부 형식을 지원하지 않고 v0·v1 경로를 서빙하지 않는다. SDK가 같다는 것이 동작이 같다는 뜻은 아니다.
10. **크롤 윤리의 전가**: robots.txt를 읽고 자신을 `FirecrawlLocal`로 식별하지만, 최종적 책임은 사용자에게 있다고 라이선스가 못 박는다. 대규모 수집의 법적·윤리적 판단은 도구가 대신하지 않는다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **폴백을 정책으로 쓰기**: 실패를 예외가 아니라 다음 상태로 모델링하고, 각 경로에 시간 상한과 우선순위를 주는 설계는 검색·RAG·에이전트 도구 호출 전반에 그대로 이식된다. "가장 좋은 엔진"을 고르는 대신 "가장 싼 엔진부터, 실패하면 올라가기"를 택한 것은 비용·지연·성공률의 3목적 절충이다.
- **출처를 1급 필드로 두기**: `source`와 `blocked` 필드는 데이터 계보(lineage)를 API 계약 안으로 끌어들인 예다. 프라이버시·감사 요구가 있는 조직에서는 이런 필드의 유무가 도입 여부를 가른다.
- **알 수 없는 값의 안전한 기본값**: 설정 문자열을 3상태로 해석하고 미지의 값을 `off`로 처리하는 패턴은, 데이터 유출 사고의 상당 부분이 "오타 난 설정"에서 온다는 사실에 대한 공학적 답이다.
- **게이트의 위치**: 프롬프트 인젝션 검사를 LLM 형식 변환 직전에 두는 것은 비용과 노출을 동시에 줄인다. 검사 위치가 곧 위협 모델이라는 점을 보여주는 배치다.
- **경계의 계층화**: CLI는 관대하게, 서버는 엄격하게, 메타데이터 주소는 항상 거부. 주체별로 다른 기본값을 두는 이 방식은 로컬 도구를 여럿 붙이는 에이전트 환경의 표준이 될 만하다.
- **후속 연구 방향**: (i) 폴백 정책의 비용·성공률 최적화(강화학습 또는 온라인 학습), (ii) 마크다운 손실률의 과제별 측정, (iii) 인젝션 가드의 재현율·오탐률 공개 평가, (iv) 부분 검사 페이지에 대한 위험 가중치 설계, (v) 로컬·클라우드 혼합 실행에서의 데이터 계보 표준화.
