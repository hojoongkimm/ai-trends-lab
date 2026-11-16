# 🛠️ Stitch Skills: Design-to-Code as a Gate-Enforced Skill Library — Plugin-Bundled Agent Skills, a Machine-Checked Rule Subset, and File-Borne Loop State

- **Repository**: [google-labs-code/stitch-skills](https://github.com/google-labs-code/stitch-skills)
- **Stars**: ⭐ 9,200+ (forks 1,100+, created 2026-01, last commit 2026-08)
- **Category**: `Skills` / Design-to-Code Skill Library & Gate-Enforced Generation
- **Organization**: Google Labs Code (labeled not an officially supported Google product)
- **License / Runtime**: Apache-2.0 / TypeScript and JavaScript skill resources with Node.js scripts (`@swc/core` AST validator, Puppeteer snapshot, `tsx` extras), 3 plugins, 16 `SKILL.md` files across 88 repository files (about 900 KB), installable through the Codex marketplace, `npx plugins`, and manual copy for OpenCode
- **Key Themes**: plugin-packaged Agent Skills, four-phase gates with mandatory checkpoints, a validator that machine-checks two of the stated architectural rules, design-token separation between project-level theme and generation prompt, `SITE.md` and `DESIGN.md` as durable conventions, a baton file that moves loop state out of context, and CI that only re-validates the exemplar

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

생성형 디자인 도구가 만든 화면을 실제 코드로 옮기는 작업은 "코드를 생성하는가"보다 **관례를 유지하는가**의 문제다. 반복 실행에서 관찰되는 병목은 세 가지다.

1. **반복 사이의 상태 소실(State Loss Across Iterations)**: 에이전트의 컨텍스트는 매 세션 초기화된다. 사이트의 이름, 보이스, 색, 타이포, 페이지 목록이 매번 다시 추론되면 회차마다 다른 사이트가 만들어진다. 관례를 어딘가에 **기록**해 두지 않으면 드리프트는 필연이다.
2. **규범과 검사 가능성의 간극(Normativity vs Checkability)**: "모놀리식 페이지 파일 금지", "이벤트 핸들러는 훅으로", "다크 모드 변형 필수" 같은 규칙은 산문으로는 선언할 수 있어도 보장할 수 없다. 모델이 규칙을 읽었다는 것과 규칙이 지켜졌다는 것은 다른 명제다.
3. **도구 표면의 결합(Tool-Surface Coupling)**: Stitch MCP의 도구 접두사, 화면 메타데이터의 다운로드 URL, Google CDN이 기본으로 주는 저해상 썸네일, 인증이 걸린 GCS 리다이렉트 같은 세부가 작업 성패를 좌우한다. 지식과 도구 사용법이 분리되어 있으면 매번 실패한다.

이 라이브러리의 응답은 세 층의 분리다. 관례는 **파일**(`SITE.md`, `DESIGN.md`, `style-guide.json`)로 외부화하고, 규칙은 **게이트**로 단계에 묶으며, 도구 사용법은 **스킬 본문과 스크립트**로 함께 배포한다. 이 중 세 번째가 가장 성숙하고, 두 번째가 가장 야심적이며, 두 번째의 실제 강제력은 4절에서 따진다.

---

## 2. 아키텍처 및 수학적 정식화 (Architecture & Formulation)

### A. 산출물 파이프라인과 상태 벡터

디자인에서 코드까지의 경로는 네 개의 지속 산출물로 이어진다. `.stitch/SITE.md`는 프로젝트 헌법(정체성, 비주얼 언어, 파일 구조, 사이트맵, 백로그, 창작 자유도)이고, `.stitch/DESIGN.md`는 디자인 시스템(색·타이포·컴포넌트·모션·금지 패턴)이며, `resources/style-guide.json`은 현재 프로젝트의 HTML `<head>`에서 추출한 토큰이고, `.stitch/metadata.json`은 프로젝트 id·기기 유형·화면 목록·`Last Sync Time`을 담는다.

$$
S_t = \bigl(\mathrm{SITE},\ \mathrm{DESIGN},\ G_t,\ M_t,\ C_t\bigr)
$$

$G_t$는 스타일 가이드, $M_t$는 메타데이터, $C_t$는 컴포넌트 집합이다. `Last Sync Time`은 이 벡터에 버전 토큰을 붙이는 장치다. 코드가 어느 시점의 디자인에서 나왔는지 알 수 있으므로, 재동기화가 필요한지 판정할 수 있다. 이 저장소가 상태를 컨텍스트가 아니라 파일에 두는 선택은 반복 실행에서 결정적이다. 세션 $t$의 컨텍스트 비용이 $O(1)$로 유지되고, 누적 상태는 파일 크기 $O(\lvert S \rvert)$로만 증가한다.

### B. 게이트: 단계에 묶인 통과 조건

`react-components` 스킬은 4단계로 나뉘고 각 단계에 GATE가 붙는다. 1단계는 모든 화면을 MCP로 가져와 스크립트로 내려받고 스크린샷을 **눈으로 확인**할 때까지 끝나지 않는다. 2단계는 현재 프로젝트의 토큰으로 스타일 가이드를 덮어쓸 때까지, 3단계는 컴포넌트 아키텍처 규칙을 모두 만족할 때까지다. 게이트는 "다음 단계로 가기 전에 참이어야 하는 명제"이고, 각 단계에는 안티패턴 목록이 붙어 실패 모드를 명시한다.

문제는 4단계다. 검증·감사·브라우저 테스트는 **선택**이며 사용자 허락을 먼저 받으라고 되어 있다. 즉 실행 가능한 게이트가 마지막 단계에서 스킵될 수 있다. 게이트의 강제력이 단계마다 다르다는 뜻이고, 이는 4절 4항의 핵심 지적이다.

### C. 기계 검사되는 규칙의 부분집합

3단계가 선언하는 아키텍처 규칙은 여덟 가지다(모듈형 컴포넌트, 로직 격리, 데이터 분리, 타입 안전, 프로젝트 적합성, 내비게이션 배선, 스타일 매핑, 다크 모드). 그중 실행 가능한 검사기는 두 개만 본다. `scripts/validate.js`는 `@swc/core`로 AST를 파싱해 다음 술어를 평가한다.

$$
V(f) = \bigl[\exists I \in \mathrm{Intf}(f) : \mathrm{name}(I) \text{ ends with Props}\bigr] \;\wedge\; \bigl[\nexists a \in \mathrm{JSXAttr}(f) : \mathrm{className}(a) \ni \mathrm{hex}\bigr]
$$

첫 항은 `Props`로 끝나는 인터페이스 선언의 존재, 둘째 항은 `className` 안의 16진 색상 부재다. 검사기는 82줄이고, 실패 시 종료 코드 1을 낸다. 규칙 집합 $R$과 검사 집합 $V$의 비율은 다음과 같다.

$$
\frac{\lvert V \rvert}{\lvert R \rvert} = \frac{2}{8} = 0.25
$$

나머지 여섯 규칙(모듈 분해, 훅 분리, 목 데이터, `href="#"` 치환, 테마 클래스 사용, 다크 모드 변형)은 산문으로만 존재한다. 이 비율이 이 라이브러리의 신뢰도를 가장 정직하게 요약한다. **선언된 품질의 4분의 1만 기계로 확인된다.**

### D. 토큰의 단일 소유자: 생성 프롬프트에서 테마를 빼는 규칙

`generate-design`은 생성 전에 프롬프트를 반드시 보강하도록 요구하고, 그 과정에서 색·폰트·라운드니스 같은 디자인 토큰을 **생성 프롬프트에서 제외**한다. 토큰은 프로젝트 수준의 디자인 시스템이 적용하며, 프롬프트에 중복되면 충돌한다. 편집 프롬프트에서는 정밀 조정을 위해 16진 색상을 허용한다.

$$
G \cap T = \varnothing \quad \text{(생성)}, \qquad G \cap T \neq \varnothing \ \text{허용} \quad \text{(편집)}
$$

이 비대칭은 미학이 아니라 일관성 공학이다. 같은 속성에 두 개의 쓰기 주체가 있으면 마지막 쓰기가 이기고, 그 결과가 확률적 생성이라면 충돌은 재현 불가능한 형태로 나타난다. **테마는 프로젝트가, 레이아웃은 프롬프트가 소유**한다는 단일 쓰기 원칙은 다른 생성 파이프라인에도 그대로 옮길 수 있다.

### E. 배턴 루프: 상태를 컨텍스트 밖으로

`stitch-loop`는 한 번에 한 페이지를 만들고 다음 회차의 지시를 파일에 남기는 릴레이 패턴이다. 회차 $t$는 `.stitch/next-prompt.md`를 읽어 페이지 이름과 프롬프트를 얻고, 생성·통합한 뒤 다음 배턴을 쓴다.

$$
s_{t+1} = \Phi\bigl(s_t,\ b_t\bigr), \qquad b_t = \texttt{next-prompt.md}
$$

배턴에는 YAML 프론트매터의 `page`(출력 파일명)와 `DESIGN.md` 6절의 디자인 시스템 블록이 반드시 포함된다. 그래서 컨텍스트가 초기화되어도 회차 간 정보 손실이 파일 경계에서만 발생하고, 그 손실은 사람이 읽고 고칠 수 있는 형태다. 반복 길이가 $n$일 때 컨텍스트 비용은 $O(1)$로 유지되지만, 배턴의 품질이 곧 다음 회차의 품질이 된다. 상태 외부화의 대가다.

### F. 추출의 충실도: 계산된 스타일과 자기완결 HTML

`code-to-design`과 `extract-static-html`은 실행 중인 앱에서 정적 HTML을 뽑는다. 두 전략이 있고, 스킬은 사용자에게 선택을 요구하며 Puppeteer 쪽을 기본으로 권한다. Puppeteer 경로는 헤드리스 Chrome으로 렌더링된 DOM을 잡고 CSS를 인라인하며 이미지를 base64 데이터 URI로 바꾼다. 핵심은 충실도다. 계산된 스타일(computed style)이 해석된 상태를 저장하므로 CSS 변수·프레임워크 유틸리티·런타임 주입 스타일이 모두 굳는다. 대신 크기가 커진다. 원본 HTML 바이트 $H$, 이미지 바이트 $I$, 인코딩 팽창 $\alpha \approx 4/3$에 대해,

$$
\lvert \mathrm{out} \rvert \approx H + \alpha \sum_{k} \lvert I_k \rvert
$$

브라우저 서브에이전트 경로는 상호작용이 필요한 인증 벽 뒤에서 유리하지만, 결과가 에이전트 컨텍스트를 지나므로 잘릴 수 있다. 두 경로의 차이는 "충실도 대 컨텍스트 전달 비용"이라는, 이 저장소에서 반복되는 교환이다.

### G. 배포 표면: 플러그인과 마켓플레이스

저장소는 `plugins/` 아래 3개의 플러그인(`stitch-design` 6개, `stitch-build` 5개, `stitch-utilities` 4개 스킬, 그리고 문서화되지 않은 `site-md` 1개)으로 구성되고, `.agents/plugins/marketplace.json`이 설치 정책(`AVAILABLE`, `ON_INSTALL` 인증)과 카테고리를 선언한다. Codex는 마켓플레이스와 `--sparse` 체크아웃으로, Claude Code와 Cursor는 `npx plugins`로, OpenCode는 수동 복사로 설치한다. OpenCode는 스킬 이름이 소문자 kebab-case여야 하고 `stitch::` 접두사를 허용하지 않아 이름과 폴더를 바꿔야 한다.

설치 표면이 늘어난 만큼 이식성의 균열도 늘어난다. `allowed-tools` 같은 프론트매터는 일부 호스트가 무시하고, 도구 이름(`web_fetch` 대 `webfetch`)이 호스트마다 다르며, 스킬 간 의존(`generate-design`이 `manage-design-system`을 먼저 요구)이 있어 선택 설치가 조용히 깨질 수 있다. README가 "의존성을 함께 설치하라"고 경고하는 이유다.

### H. 검증 자산: 자기가 만든 예시를 다시 검사하는 CI

`.github/workflows/validate-skills.yml`은 두 작업으로 `react-components`와 `react-native`의 `examples/gold-standard-card.tsx`에 검사기를 돌린다. 즉 CI가 확인하는 것은 **검사기가 예시를 통과시키는가**이지, 생성된 코드가 규칙을 지키는가가 아니다. 검사기와 예시가 같은 사람에 의해 함께 작성되었으므로 이 검사는 자기 일관성 검사다. 회귀 방지로서는 유효하지만, 품질 보증으로 읽으면 과대해석이다.

---

## 3. 구조적 비교 (Comparative Positioning)

| 항목 | 일회성 코드 생성 프롬프트 | 프레임워크 스캐폴더 | 사람의 디자인 핸드오프 | Stitch Skills |
| :--- | :--- | :--- | :--- | :--- |
| 반복 사이 관례 | 매번 재추론 | 없음 | 문서·회의 | `SITE.md`, `DESIGN.md`, 토큰 파일 |
| 상태 소유자 | 컨텍스트 | 없음 | 사람 | 파일 + 배턴 |
| 규칙 강제 | 없음 | 생성기 코드 | 리뷰 | 게이트 + 부분 검사기 |
| 기계 검사 규칙 | 없음 | 구조 전체 | 없음 | 8개 중 2개 |
| 도구 통합 | 없음 | 없음 | 없음 | Stitch MCP + 스크립트 |
| 설치 표면 | 1개 | 1개 | 없음 | 3플러그인 · 5하네스 |
| 품질 증거 | 없음 | 테스트 | 리뷰 기록 | 예시 검사(자기 일관성) |

`skills` 카테고리 안에서 이 라이브러리의 위치는 "규격(Anthropic) → 방법론(Superpowers) → 도메인 지식(Claude-BugHunter) → **산출물 파이프라인**"의 마지막 칸이다. 다른 문서들이 지식이나 규율을 공급하는 반면, 이쪽은 지식·스크립트·게이트·상태 파일을 한 묶음으로 배포해 **한 번의 산출물**을 만든다. 강점은 도구 사용법과 산출물 규약이 함께 온다는 점이고, 약점은 그 규약의 강제력이 산문과 2개 술어에 머문다는 점이다.

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **검사 가능한 규칙이 4분의 1**: 여덟 규칙 중 둘만 기계로 확인된다. 모듈 분해·훅 분리·목 데이터·라우팅 배선·다크 모드 변형은 검사되지 않으며, 이들은 실제로 유지비가 가장 큰 규칙들이다. 검사기가 통과했다는 사실은 "Props 인터페이스가 있고 16진 색상이 없다"만 뜻한다.
2. **검사기의 얕은 판정**: `Props` 접미사 이름만 보고 타입의 적합성이나 `Readonly` 여부를 확인하지 않으며, 색상 검사는 `className` 문자열 안의 16진 패턴만 본다. 테마 토큰이 아닌 임의의 Tailwind 임의값(`bg-[#123456]`)이나 스타일 객체는 다른 경로로 통과한다.
3. **CI의 순환성**: 워크플로는 생성된 코드가 아니라 예시 파일을 검사한다. 생성 결과의 품질을 재는 테스트가 없으므로, 규칙 위반이 실제로 얼마나 발생하는지는 측정되지 않는다.
4. **마지막 게이트가 선택 사항**: 4단계의 검증·감사·브라우저 테스트는 사용자 허락을 받아 진행하도록 되어 있어, 실행 가능한 검사가 가장 필요한 지점에서 스킵될 수 있다. 게이트가 강제가 아니라 권고로 퇴화하는 전형적 경로다.
5. **산문 압박에 의존하는 준수**: "MANDATORY", "PROHIBITED" 같은 대문자 지시는 모델의 순응을 구하는 방식이다. 준수 여부는 호스트·모델·컨텍스트 길이에 따라 달라지고, 강제 계층이 없다.
6. **유지보수의 정체**: 마지막 커밋이 2026-08로, MCP 도구 표면이 빠르게 바뀌는 영역에서 석 달 넘게 정지해 있다. 저장소도 "공식 지원 제품이 아니다"라고 밝힌다. 도구 접두사·다운로드 URL·CDN 동작이 바뀌면 스크립트와 본문이 함께 낡는다.
7. **문서와 실제의 불일치**: README는 유틸리티 4개를 나열하지만 저장소에는 `site-md`가 더 있어 스킬이 15개가 아니라 16개다. 설치 안내와 카탈로그가 실제 자산과 어긋나는 것은 스킬 코퍼스에서 흔한 부패 형태다.
8. **이식성의 균열**: 호스트마다 스킬 이름 규칙, 프론트매터 해석, 도구 이름이 다르다. `stitch::` 접두사는 OpenCode에서 로드되지 않고, `allowed-tools`는 무시되며, `web_fetch`/`webfetch` 차이는 본문 수정 없이는 해결되지 않는다.
9. **컨텍스트 비용의 비가시성**: 스킬마다 `resources/`, `examples/`, 스크립트가 붙고 의존 사슬이 있어, 실제 로드되는 총량은 선택적 설치에서 예측하기 어렵다. 저장소 어디에도 스킬별 토큰 크기 표가 없다.
10. **미학적 주장의 측정 부재**: `taste-design`은 "프리미엄, 안티제너릭"을 밀도·분산·모션·창의성의 1–10 스칼라와 금지 패턴으로 조작화하지만, 그 결과가 실제로 더 나은 디자인인지 재는 평가는 없다. 조작화는 진전이고 측정은 부재다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **스킬을 컴파일러로 보는 관점**: 규범 명세(산문), 실행 가능한 부분 검사(술어), 정답 예시(골드 스탠다드)의 3층 구성은 컴파일러의 타입 검사와 유사하다. 이때 검사 커버리지, 즉 $\lvert V \rvert / \lvert R \rvert$는 스킬 팩의 신뢰도를 비교하는 지표가 된다. 0.25라는 값은 이 라이브러리의 자기 서술보다 많은 것을 말해준다.
- **상태 외부화와 배턴 패턴**: 반복 실행에서 컨텍스트 대신 파일을 기억으로 쓰는 설계는, 긴 작업을 여러 세션으로 쪼개는 모든 파이프라인에 적용된다. 회차 간 정보 손실이 파일 경계에서만 일어나고 사람이 개입할 수 있다는 점이 핵심이며, 배턴 내용의 충실도가 성능 변수라는 것도 실험 가능한 명제다.
- **단일 쓰기 원칙으로서의 토큰 분리**: 생성 프롬프트에서 테마를 빼고 프로젝트 수준에 두는 규칙은, 같은 속성의 다중 쓰기 주체가 만드는 비결정성을 제거한다. 이 원칙은 프롬프트 엔지니어링이 아니라 일관성 공학의 문제로 다뤄야 한다.
- **계산된 스타일의 가치**: 정적 HTML 추출에서 충실도를 결정하는 것은 DOM이 아니라 해석된 스타일이다. 디자인 산출물을 기계가 다루는 과업(회귀 비교, 시각 차분, 재생성)에서 이 구분은 반복적으로 중요해진다.
- **부분 검사의 정직한 표기**: 무엇이 검사되고 무엇이 검사되지 않는지를 문서에 적는 것만으로도 사용자는 위험을 계산할 수 있다. 이 저장소는 그 표기를 하지 않지만, 본 문서의 표가 그 자리를 대신한다.
- **후속 연구 방향**: (i) 게이트 커버리지와 실제 결함률의 관계, (ii) 반복 회차에 따른 관례 드리프트의 정량화(스타일 토큰 일치율, 컴포넌트 재사용률), (iii) 호스트 간 도구 이름 추상화 계층의 설계, (iv) 배턴 충실도와 산출물 품질의 상관, (v) 미학적 제약을 측정 가능한 지표로 바꾸는 방법.
