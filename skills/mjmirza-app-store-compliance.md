# 🛡️ app-store-compliance: Policy Knowledge as a PreToolUse Gate — 123 Rejection Predicates, a Dated Regulatory Ledger, and a Fail-Closed Submit Guard

- **Repository**: [mjmirza/app-store-compliance](https://github.com/mjmirza/app-store-compliance)
- **Stars**: ⭐ 1,864 (forks 126, created 2026-06-06, last push 2026-11-19)
- **Category**: `Skills` / Compliance Corpus and Pre-Submission Enforcement Layer
- **Organization**: Mirza Iqbal / next8n (single maintainer; licence reported as `NOASSERTION` by GitHub, with an OpenRoots ORA 2.3 badge in the README)
- **License / Runtime**: non-standard open licence / Bash for the guard and installer, Python 3 for the audit and monitor scripts, JSON for the rule data; no runtime service and no network dependency for the scan itself
- **Key Themes**: 123 machine-readable rejection patterns with detection signals and counter-signals, 107 regulatory deadlines carrying jurisdiction, effective, grace and enforcement dates plus a verified-on date and source URL, a 1,752-line guard that hooks `PreToolUse` and blocks submit commands with exit code 2, a three-valued verdict (blocked, clear, not checked) that never reports an unfinished scan as clean, and an installer that proves itself by feeding a submit command through each host's own payload format

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

앱 심사 거절의 구조는 일반적인 버그와 다르다. 실패의 단위가 코드 결함이 아니라 **심사자가 적용하는 정책 술어**이고, 피드백은 한 줄이며, 주기는 주 단위다. 제출 → 대기 → 한 줄 거절 → 추측 수정 → 재제출의 루프에서 한 바퀴의 비용이 일주일이다.

1. **규칙의 이동성(Rule Drift)**: 가이드라인, 데이터 보호 규칙, 지역 규제가 계속 바뀐다. 작년에 통과한 앱이 올해 거절되는 것은 예외가 아니라 기본값이다.
2. **코드 밖 원인(Out-of-Code Causes)**: 거절의 상당 부분은 소스에 없다. 개발자 계정 상태, 동의한 계약, 데모 계정, 스토어 메타데이터, 개인정보 처리방침 URL처럼 콘솔과 계정에 있는 것들이다. 코드 스캐너는 여기를 볼 수 없다.
3. **생성 속도와 심사 속도의 역전(Generation Outpaces Review)**: 에이전트가 앱을 몇 시간에 만들면서 제출량이 늘었고, 규칙은 느슨해지지 않았다. 사람이 읽는 체크리스트만으로는 속도를 따라갈 수 없다.
4. **주장의 검증 부재(Unverified Claims)**: 컴플라이언스 자료는 틀린 인용으로 가득하기 쉽다. 조항 번호와 날짜가 틀리면 사용자는 잘못된 결정을 내린다.

응답은 **정책을 술어로 바꾸고, 술어를 실행 게이트로 만들고, 게이트가 볼 수 없는 것은 통과가 아니라 미확인으로 보고한다**이다.

---

## 2. 아키텍처 및 정식화 (Architecture & Formulation)

### A. 정책의 술어화: 신호와 반대 신호

`data/rejection-patterns.json`(v2.4.0, 2026-11-15 갱신)은 123개 패턴을 담는다. 각 패턴은 플랫폼, 인용 조항 또는 정책 이름, 심각도, 탐지 설명, 신호 목록, 필요 시 반대 신호 목록, 그리고 수정 지침을 갖는다.

$$
\text{fire}(p, T) = \text{signal}(p, T) \land \lnot \text{counter}(p, T)
$$

반대 신호가 이 스키마의 핵심이다. "계정 생성이 있는데 삭제 경로가 없다"는 패턴은 `signUp` 신호와 `deleteAccount` 반대 신호의 곱이다. 반대 신호가 없으면 모든 로그인 앱이 삭제 기능 누락으로 잘못 걸린다. 즉 이 데이터는 키워드 목록이 아니라 **부정 조건을 포함한 판정식**이다.

| 구분 | 수 | 비고 |
| :--- | :---: | :--- |
| 전체 패턴 | 123 | v2.4.0 |
| 플랫폼: apple / google / both | 61 / 43 / 13 | 웹 6 |
| 심각도: critical / high / medium | 38 / 65 / 20 | 게이트 차단은 critical만 |
| 탐지 레시피 | 113 | 패턴별 재현 가능한 명령 |
| 규제 데드라인 | 107 | 관할 20여 개 |
| 판정 등록부 항목 | 14 | 조항 해석과 근거 |

### B. 게이트의 판정 함수

가드는 `PreToolUse` 훅으로 등록되어 셸 명령을 받아, 제출 명령(`fastlane`, `eas submit`, `gradle bundleRelease`, `xcrun altool` 등)인지 판정하고 프로젝트를 스캔한다. 결론은 세 값 중 하나다.

$$
\text{verdict} = \begin{cases} \text{BLOCKED}, & N_{\text{crit}} > 0 \\ \text{NOT CHECKED}, & \text{scan incomplete} \lor \text{no project} \\ \text{CLEAR}, & \text{otherwise} \end{cases}
$$

차단은 종료 코드 2로 전달되고, 호스트는 그 코드로 명령 실행을 멈춘다. 예외는 환경 변수 하나(`APP_STORE_GUARD_OK=1`)로만 열린다. 중요한 것은 세 번째 값의 존재다. 임시 버퍼를 만들지 못하면 결과는 CLEAR가 아니라 NOT CHECKED이고, 종료 코드는 1이다. iOS·안드로이드 프로젝트를 찾지 못해도 같은 값이 나온다. 스캔이 끝나지 않은 상태를 통과로 표시하지 않는 이 규칙이, 자동화 게이트에서 가장 흔한 실패(빈 결과를 깨끗함으로 반환)를 막는다.

가드는 1,752줄이고, 판정 항목은 100개다(critical 29, high 48, medium 23). 플랫폼 판별은 파일 존재로 하고(iOS, Android, Flutter, React Native/Expo, Ionic/Capacitor, Unity, Tauri, KMP, MAUI, 웹), 크로스플랫폼은 네이티브 심볼이 소스에 나타나지 않으므로 플러그인 패키지명으로 우회한다. 예를 들어 인앱 결제 검사는 `in_app_purchase`, `react-native-iap`, `Purchases_flutter` 같은 플러그인 식별자를 본다.

### C. 볼 수 있는 것과 없는 것의 분리

스캔은 소스·설정·매니페스트를 읽는다. 읽을 수 없는 것이 명시되어 있다. 개발자 계정의 활성 상태, 동의한 계약, App Store Connect에 입력한 심사 노트와 데이터 사용 선언, 데모 계정의 실제 동작, 스토어 메타데이터의 승인 상태다. 이 항목들은 스캔이 "깨끗함"을 주장하지 않고 **확인 필요**로 남긴다.

이 비대칭은 비용 구조에서 나온다. 놓친 경고의 비용 $C_{\text{miss}}$는 심사 한 바퀴(주 단위)이고, 잘못 띄운 경고의 비용 $C_{\text{flag}}$는 1분이다. 기대 비용을 최소화하는 판정은 둘의 비율이 클수록 더 많이 띄우는 쪽이다.

$$
E\left[\text{cost}\right] = \alpha C_{\text{miss}} + \beta C_{\text{flag}}, \qquad C_{\text{miss}} \gg C_{\text{flag}}
$$

프로젝트는 이 결론을 문서에 그대로 적는다. "확인할 수 없는 항목은 깨끗하다고 가정하지 않고 드러낸다. 놓친 경고는 거절이고, 드러낸 경고는 1분이다." 사용자 후기에서도 이 동작이 오탐처럼 보였다가, 오히려 그 덕에 제출 전 점검이 습관이 되었다는 반응이 인용된다.

### D. 날짜가 붙은 규제 장부

`data/regulatory-deadlines.json`은 107개 항목을 담고, 각 항목은 관할, 법령, 요구사항, 발효일, 유예기간, 의무화일, 집행일, 우선순위, 원문 URL, 확인일을 갖는다. 관할 분포는 유럽연합 22, Google Play 전역 16, Apple 전역 11, 미국 연방 8, 영국 7, 캘리포니아 6, 호주 4, 한국 4, 브라질 3, 중국 3 순이다.

이 장부의 설계에서 눈에 띄는 것은 **날짜를 네 개로 쪼갠 것**과 **확인일을 데이터에 넣은 것**이다. 규제는 발효와 집행이 다르고, 유예가 끼면 실무 마감이 또 달라진다.

$$
\mathrm{due}(t, J) = \min \left[ d_{\mathrm{mandatory}} \mid d \in D_{J}, d_{\mathrm{mandatory}} \ge t \right]
$$

배송 지역 집합 $J$에 대해 이 함수의 값을 90일 창으로 잘라 보여주는 것이 `deadline-checker.py`의 역할이다. 그리고 항목마다 "이 조항은 의무가 아니라 실무 증거에 가깝다" 같은 주석을 달아, 법적 의무와 권장 관행을 문장 수준에서 구분한다.

### E. 주장의 검증을 CI로 옮기기

컴플라이언스 문서의 최대 위험은 그럴듯한 오인용이다. 그래서 이 저장소는 주장 자체를 검사 대상으로 삼는다.

| 장치 | 하는 일 |
| :--- | :--- |
| `verify-citations.py` | 원문 URL이 살아 있는지가 아니라 **주장을 실제로 뒷받침하는지** 확인(네트워크 필요) |
| `.citation-allowlist` | 인용 가능한 출처 목록을 고정 |
| `.banned-words-allow` | 과장 어휘를 금지(예: "unveils") |
| `validate.py` | 세 데이터 파일의 내부 정합성 검사, CI 실행 |
| `evals/gold-cases.json` | 판정 회귀 평가용 골드 케이스 |
| `eval-scorecard.py` | 골드 케이스 대비 점수 산출 |

즉 "정책 데이터", "인용", "표현"이 모두 기계 검사 대상이다. 이 조합은 규제 문서를 소프트웨어처럼 다루는 방식이다.

### F. 설치와 유지보수의 증명

`scripts/install.sh`는 12개 이상의 에이전트 호스트(Claude Code, Codex, Gemini CLI, OpenCode, Cursor, Copilot CLI·VS Code·클라우드 에이전트, Grok, Qwen Code, Kimi Code, Factory Droid, Auggie, Windsurf)에 훅과 스킬을 심는다. 여기서 특이한 점은 **설치 성공을 파일 존재로 판정하지 않는다**는 것이다. 각 호스트의 실제 페이로드 형식으로 제출 명령을 훅에 먹여 차단되는 것을 확인한 뒤에야 `Install verified.`를 출력한다. Node가 없는 OpenCode에서는 `Installed, but not fully verified.`로 구분한다. 설정 파일은 유효하지 않은 JSON이면 절대 덮어쓰지 않고 붙여넣을 블록을 대신 제공하며, 기존 파일은 날짜가 붙은 백업을 남긴다.

모니터 계층도 같은 태도다. `monitor.py`는 Apple 개발자 RSS에서 25개 추적 영역의 변화를 읽고, 규제·프라이버시·안드로이드·보안·AI 정책별 모니터가 따로 있으며, `fleet-sweep.py`는 여러 저장소를 한 번에 훑는다. 피드에 닿지 못하면 그 항목을 "표본"이라고 밝힌다.

---

## 3. 구조적 비교 (Comparative Positioning)

| 항목 | 블로그 체크리스트 | 정적 린터 | 스토어 자체 사전 점검 | 일반 컴플라이언스 SaaS | app-store-compliance |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 기계 판독 규칙 | ✗ | 부분 | ✗ | ✓ (비공개) | ✓ (공개 JSON) |
| 제출 명령 차단 | ✗ | ✗ | ✗ | ✗ | ✓ (PreToolUse, exit 2) |
| 에이전트 호스트 적용 범위 | ✗ | ✗ | ✗ | ✗ | 12개 이상 |
| 규제 마감 장부 | 부분 | ✗ | ✗ | ✓ | ✓ (107건, 관할별) |
| 인용의 기계 검증 | ✗ | ✗ | ✗ | ✗ | ✓ |
| 미확인 항목의 명시 | ✗ | ✗ | ✗ | 부분 | ✓ (3값 판정) |
| 비용 | 무료 | 무료 | 무료 | 유료 | 무료 |
| 코드 밖 항목(계정·계약) | ✓ (사람) | ✗ | 부분 | 부분 | 사람 체크리스트로 이관 |

`skills/` 안에서의 위치는 "도메인 지식 번들"이면서 동시에 "실행 경계"다. 지식을 주입하는 스킬이 보통 산출물의 품질을 올리는 데 그친다면, 이 저장소는 명령 실행 자체를 막는 지점까지 간다.

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **정규식 탐지의 천장**: 신호와 반대 신호는 결국 패턴 매칭이다. 난독화된 코드, 생성된 코드, 설정으로 우회한 권한 선언은 놓친다. 재현율이 공개되어 있지 않다.
2. **반대 신호의 취약성**: `deleteAccount` 문자열이 있으면 삭제 경로가 있다고 본다. 함수만 있고 실제로 동작하지 않아도 통과한다. 부정 조건이 늘수록 위음성도 함께 는다.
3. **수치의 드리프트**: 저장소 설명은 117개 패턴·95개 데드라인을 말하지만, 데이터는 123개·107개다. README도 스크립트 설명에서 "95줄" 같은 옛 수치를 남긴다. 문서와 데이터의 동기화가 이 프로젝트의 상시 부채다.
4. **라이선스의 불명확성**: GitHub는 `NOASSERTION`으로 표시하고, README 배지는 OpenRoots ORA 2.3을 가리킨다. 사내 도입 전에 라이선스 조항을 직접 읽어야 한다. MIT라고 가정하면 안 된다.
5. **마케팅 톤과 증거의 혼합**: 심사 제출량·거절률·차단 건수는 언론 보도와 스토어 발표에서 온 2차 인용이다. "첫 시도 승인" 사례는 Reddit 스크린샷 한 건이다. 데이터 자체는 검증되지만 효과 주장은 일화 수준이다.
6. **정책 부패(Policy Rot)**: 107개 데드라인과 123개 패턴은 갱신 노동을 전제로 한다. 단일 유지보수자 구조에서 갱신이 멈추면 게이트는 조용히 낡은 규칙으로 통과시킨다.
7. **차단의 우회 가능성**: 환경 변수 하나로 예외가 열리고, 훅을 지원하지 않는 도구(Aider, Copilot for Xcode 등)에서는 수동 실행에 의존한다. 게이트는 실수 방지 장치이지 강제 장치가 아니다.
8. **설치기의 침습성**: 여러 도구의 설정 파일을 수정한다. 백업과 검증 절차가 있지만, 조직 정책상 설정 변경이 통제되는 환경에서는 그 자체가 장애물이다.
9. **볼 수 없는 것의 크기**: 계정·계약·콘솔 상태는 여전히 사람의 몫이다. 문서가 이를 정직하게 이관하지만, 사용자는 "스캔 통과"를 "제출 가능"으로 오독하기 쉽다.
10. **플랫폼 편중**: Apple 가이드라인이 패턴 ID의 절반 가까이(54개)를 차지한다. 안드로이드·웹·기타 스토어는 상대적으로 얕다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **정책을 부정 조건 포함 술어로**: "신호가 있고 반대 신호가 없을 때"라는 형태는 컴플라이언스뿐 아니라 보안 규칙(SIEM), 코드 리뷰 규칙, 데이터 거버넌스 규칙에도 그대로 옮길 수 있는 표현이다.
- **3값 판정과 fail-closed**: 통과·차단의 2값 체계는 미확인을 통과로 밀어 넣는다. 미확인을 별도 값으로 두고 종료 코드로 구분하는 설계는 자동화 게이트의 기본형이 되어야 한다.
- **비대칭 비용에 맞춘 임계값**: 놓침과 오탐의 비용 차이가 크면 임계값을 낮추는 것이 최적이다. 이 판단을 문서에 명시하면 오탐이 버그가 아니라 설계로 이해된다.
- **날짜를 쪼갠 규제 장부**: 발효·유예·의무화·집행을 분리하고 확인일을 데이터에 넣는 구조는, 어떤 규제 추적 시스템에도 필요한 최소 스키마다.
- **인용 검증의 CI화**: "출처가 살아 있는가"가 아니라 "출처가 주장을 지지하는가"를 검사하는 단계는, LLM이 만든 문서가 늘어날수록 중요해진다.
- **설치의 증명**: 파일이 생겼는지가 아니라 실제 페이로드로 차단이 일어나는지를 확인하는 설치 검증은, 호스트마다 형식이 다른 에이전트 생태계에서 재사용 가능한 패턴이다.
- **후속 연구 방향**: (i) 골드 케이스 기반 재현율·오탐률 공개 측정, (ii) RSS·법령 피드에서 규칙 후보를 추출하고 사람이 승인하는 반자동 갱신, (iii) 코드 밖 항목의 API 연동(App Store Connect·Play Developer API)으로 미확인 영역 축소, (iv) 규칙 간 충돌 탐지, (v) 판정 근거의 설명 가능성.
