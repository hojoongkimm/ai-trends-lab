# 📱 Phone Harness: One Op Vocabulary over Mirroring, adb and Cloud Phones — Screen-as-API, Coordinate Mapping, and End-of-List by Movement

- **Repository**: [phone-harness/phone-harness](https://github.com/phone-harness/phone-harness)
- **Stars**: ⭐ 5,820 (forks 615, created 2026-08-07, last commit 2026-11-24)
- **Category**: `Harness` / Mobile Device Control Harness & Substrate Abstraction
- **Project**: phone-harness; PyPI package `phone-harness` 0.3.3; site and hosted service at phone-harness.com; MCP endpoint `https://mcp.phone-harness.com/mcp`; Hermes and OpenClaw plugins under `integrations/`
- **License / Runtime**: MIT / Python ≥3.10 (setup prompt pins 3.12 through `uv`), about 8,700 lines under `src/phone_harness`; macOS-only dependencies (Quartz, Vision, ApplicationServices) are skipped off macOS, where Android over adb still runs
- **Key Themes**: a 27-op device vocabulary as the only shared contract, screen-as-API with an explicit `source` field (`tree` versus `pixels`), image-pixel to global-screen-point mapping, verification-first action loop, end-of-list decided by whether the screen moved, background input through SkyLight event records with focus accounting, cloud phones reached over HTTPS instead of adb

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

에이전트가 다룰 수 있는 실행 표면은 지난 3년 동안 계속 넓어졌다. 셸, 파일 시스템, 브라우저, GUI 자동화, 그리고 최근에는 데스크톱 애플리케이션의 상태 트리까지 표준화됐다. 그런데 사용자가 실제로 하루 종일 쓰는 기기인 **스마트폰**만은 이 목록에서 빠져 있었다. 이유는 세 가지가 겹치기 때문이다.

1. **관측 채널이 없다.** iOS에는 사용자 영역의 입력 주입 API가 없고, 탈옥 없이는 프로세스를 띄울 수 없다. macOS의 iPhone Mirroring 창은 화면을 **비디오 스트림**으로 렌더링하므로 접근성 트리(accessibility tree)가 비어 있다. 즉 기기 안쪽에서 상태를 물어볼 방법이 없고, 남은 것은 픽셀과 OCR뿐이다. Android는 반대 극단으로, `adb`라는 단일 명령 채널과 `uiautomator` 접근성 트리가 있어 문자열과 경계 상자가 정확히 나온다.
2. **조작 채널이 비대칭이다.** Android는 `input tap`이라는 좌표 명령이 있고, iOS는 그런 것이 없다. 미러링 창에는 HID 수준의 합성 이벤트(CGEvent)를 쏴야 하는데, 이벤트는 **전면 창(frontmost)** 으로만 전달되므로 자동화가 수행되는 동안 사용자의 화면을 빼앗는다. 사용자가 다른 앱을 쓰고 있는 동안 몰래 조작하려면 창 서버의 이벤트 레코드 경로를 따로 구현해야 한다.
3. **실패가 침묵한다.** 웹에는 DOM이 있고 CLI에는 종료 코드가 있지만, 폰에서는 탭이 빗나가도 예외가 나지 않는다. 포커스가 없는 입력창에 텍스트를 보내면 어디로도 가지 않고 조용히 사라진다. "성공했다"는 반환값이 존재하지 않으므로, 성공 판정은 전적으로 관측에서 다시 세워야 한다.

여기에 네 번째 병목이 붙는다. **연결 자체가 사용자의 손에 있다.** 페어링, 잠금 해제, "허용" 탭, 클라우드 로그인 승인은 대행할 수 없다. 따라서 하네스는 "연결할 수 없음"과 "명령이 실패함"을 구분해서, 사용자에게 무엇을 요청해야 하는지 문장으로 돌려줘야 한다.

이 저장소의 설계 주장은 이 네 병목을 **하나의 어휘**로 흡수하겠다는 것이다. 서로 다른 네 종류의 폰(iPhone 미러링, 데스크톱 Android, 클라우드 Android, HTTPS 클라우드 iPhone)이 같은 헬퍼 함수로 조작되고, 다른 점은 각 백엔드 안에만 존재한다.

---

## 2. 아키텍처 및 수학적 정식화 (Architecture & Formulation)

### A. 유일한 이음새: 연산 어휘와 `Unsupported`

구조는 의도적으로 얇다. 모든 백엔드는 `send(op, **kw)` 하나만 구현하고, 어휘는 `transport.py`의 27개 연산 이름으로 고정된다.

| 계열 | 연산 | 의미 |
| :--- | :--- | :--- |
| 화면 | `screen.capture`, `screen.bounds`, `screen.require`, `screen.text`, `screen.text_pixels` | 캡처, 창 경계, 텍스트 상자 |
| 입력 | `input.tap`, `input.press`, `input.drag`, `input.scroll`, `input.keys`, `input.text` | 좌표·제스처·키·문자 |
| 내비게이션 | `nav.home`, `nav.back`, `nav.recents` | 홈·뒤로·최근 앱 |
| 앱 | `apps.launch`, `apps.current`, `apps.list` | 실행·전면 앱·설치 목록 |
| 세션 | `session.state`, `session.require`, `session.refocus`, `session.detail`, `device.wake` | 사용 가능 여부와 복구 |
| 관측 보조 | `focus.probe`, `focus.diff`, `url.open`, `tree`, `raw` | 방해 회계, 링크, 트리, 탈출구 |

연산 이름은 **플랫폼이 하는 방식**이 아니라 **의미**로 붙는다. `nav.home`은 iPhone에서는 미러링 창에 `Cmd+1`을 보내는 키 입력이고, Android에서는 `KEYCODE_HOME`이다. 계약이 같으므로 상위 헬퍼 `tap_text`, `scroll`, `scroll_collect`는 플랫폼을 전혀 모른다.

구현은 파생 규칙 하나로 디스패치된다. 연산 $o$의 처리기는 점을 밑줄로 바꾼 이름의 메서드이고, 그 메서드가 없으면 실패가 아니라 **부재**다.

| 조건 | 결과 |
| :--- | :--- |
| $o \in S(B)$ | 해당 백엔드 메서드 호출 |
| $o \notin S(B)$ | `Unsupported` 예외(재시도 무의미) |

여기서 $S(B)$는 백엔드 $B$가 실제로 구현한 연산 집합이다. 이 규칙 덕분에 백엔드는 자기 능력 목록을 코드와 따로 유지할 필요가 없고, `supports(op)`는 메서드 존재 검사와 같아진다. 중요한 것은 **에뮬레이션하지 않는다**는 정책이다. iOS에는 시스템 뒤로가기가 없으므로 `nav.back`은 없다. 가장자리 스와이프로 흉내 내면 호출자는 그것이 진짜 뒤로가기인지 아닌지 구분할 수 없다. 없는 기능을 그럴듯한 오답으로 채우지 않는 것이 이 어휘의 규율이다.

또한 `connect()`는 모듈 전역이 아니라 백엔드 객체를 돌려준다. 한 스크립트가 폰 두 대를 동시에 들고 작업할 수 있고, 기본 바인딩은 그중 하나일 뿐이다.

### B. 좌표 공간의 사상

화면에서 얻은 픽셀 좌표와 실제로 탭해야 하는 전역 화면 좌표는 세 가지 이유로 어긋난다. 창이 움직이고, 창 크기가 바뀌고, Retina 캡처와 논리 좌표의 배율이 다르다. 그래서 변환은 매 호출마다 현재 창 기하를 다시 읽어서 수행한다. 창 경계가 $(X, Y, W, H)$, 캡처 이미지가 $w_{\text{img}} \times h_{\text{img}}$일 때 이미지 점 $(p_x, p_y)$의 중심은 다음처럼 사상된다.

$$
x_{\text{screen}} = X + \frac{W}{w_{\text{img}}} \left( p_x + \frac{p_w}{2} \right)
$$

$$
y_{\text{screen}} = Y + \frac{H}{h_{\text{img}}} \left( p_y + \frac{p_h}{2} \right)
$$

Retina 배율은 $W / w_{\text{img}}$ 안에 자동으로 흡수되므로 별도 분기가 필요 없다. OCR 쪽에서는 정규화 경계 상자의 원점이 **좌하단**이고 화면 좌표는 **좌상단**이므로 y축을 한 번 뒤집는다. 이 뒤집기를 빠뜨리면 폰의 위아래가 뒤바뀐 탭이 나가고, 그것도 조용히 실패한다.

### C. 텍스트 추출의 두 충실도

`screen.text`는 화면의 모든 문자열을 탭 가능한 중심 좌표와 함께 돌려준다. 그런데 같은 계약 아래 두 종류의 답이 섞인다.

- **`source: "tree"`** — OS가 보고한 정확한 문자열과 경계. Android의 `uiautomator dump`, 그리고 캔버스·게임·일부 WebView처럼 트리가 없는 화면을 위한 `screen.text_pixels` 강제 경로가 아닌 경우.
- **`source: "pixels"`** — 인식기가 추론한 문자열. iPhone 미러링은 항상 이쪽이고, 클라우드 iPhone도 서버 측 OCR을 쓴다.

여기서 설계상 중요한 관찰이 하나 있다. **신뢰도(confidence)로는 두 출처를 구분할 수 없다.** 깨끗한 텍스트에 대해 Vision은 트리와 똑같이 1.0을 돌려준다. 그래서 출처를 별도 필드로 명시해야 했고, 이것은 단순한 메타데이터가 아니라 **인식론적 표지**다. 상위 에이전트는 문자열이 틀릴 수 있는지 여부를 이 필드로 판단한다.

Android에서도 트리가 항상 답하는 것은 아니다. `uiautomator`는 UI가 유휴 상태가 되기를 약 10초 기다린 뒤 "could not get idle state"로 거부한다. 재생 중인 동영상이나 일부 설정 화면은 영원히 유휴가 되지 않으므로, 거부 한 번의 비용이 크다. 구현은 이 거부를 **20초 동안 기억**하고 그동안은 Vision OCR로 픽셀을 읽는다(느리고 흐리지만 답은 나온다). 전이 중단이나 잘린 트리 같은 다른 실패는 짧게 재시도한다.

### D. 반환값이 없는 행동: (행동, 기대 술어) 루프

폰에는 "성공"을 알려주는 반환값이 없다. 따라서 한 걸음은 행동 $a$와 관측에 대한 술어 $\phi$의 쌍으로 정의되고, 다음 관측 $o_{t+1}$에서 $\phi(o_{t+1})$이 참일 때만 성공이다. 이 정의가 없으면 조용한 무동작(no-op)과 성공을 구분할 수 없다. 에이전트 지침 문서는 이를 문장으로 못 박는다. **행동하기 전에 무엇이 바뀌어야 하는지 이름 붙일 수 없으면, 성공과 실패를 구분할 수 없다.**

그 위에 세 가지 타이밍 도구가 얹힌다. `wait_for_text`는 기대 문자열이 나타날 때까지 폴링하고, `wait_for_app`은 전면 앱 ID를 폴링하며, `wait_stable`은 연속 캡처의 MD5가 `settle`회 동일해질 때까지 기다린다(기본 `settle=2`, 최대 6초). `wait(n)`으로 화면 로딩을 기다리는 대신 관측 가능한 조건을 기다리는 것이 규칙이다.

리스트 순회는 더 흥미롭다. 끝을 판정하는 기준이 **파서가 새 항목을 찾았는가**가 아니라 **화면이 움직였는가**다. 밀집한 리스트에서 한 줄을 놓쳤다고 해서 "끝"으로 읽히면 안 되기 때문이다. 구현은 세 겹으로 이를 방어한다.

1. **내용 대역 필터** — 창 높이의 상위 6%와 하위 8%를 잘라낸다. 상단 상태 표시줄의 시계가 1분마다 바뀌면 그것이 "움직임"으로 읽혀 끝 판정이 영원히 발화하지 않기 때문이다.
2. **정착(settle)** — 스크롤 후 두 번의 연속 읽기가 **정확히 동일한 집합**이 될 때까지 최대 2.5초 기다린다. 동등 비교라 조정할 문턱값이 없고, 절대 정착하지 않는 화면(재생 중인 영상)은 그냥 예산을 다 쓴다.
3. **정체 카운터** — `scroll_collect`는 새 항목이 0인 스크롤이 `end_after`(기본 3)회 연속이면 끝으로 본다. `max_scrolls`는 400이다. `amount < 1.0`으로 화면을 겹치게 스크롤해 행이 캡처 사이로 빠지지 않게 한다.

### E. 배경 조작: 창을 빼앗지 않는 입력

기본 백엔드는 행동 전에 미러링 창을 전면으로 올린다. 캡처가 `screencapture`, 주입이 `CGEventPost`이고 둘 다 포커스된 앱을 대상으로 하기 때문이다. 사용자 입장에서는 자동화가 도는 동안 화면을 쓸 수 없다. 이를 없앤 것이 `background.py`이고, 두 조각으로 이뤄진다.

- **눈** — `CGWindowListCreateImage`는 창 ID로 특정 창을 캡처하므로, 활성 앱이 아니거나 다른 창에 가려져 있어도 읽힌다.
- **손** — macOS가 실제로 막는 것은 앱이 *활성*인지 여부다. 일반 CGEvent는 이 경계를 넘지 못하지만, 창 서버는 프로세스에 직접 전달되는 합성 이벤트 레코드를 받는다(`SLPSPostEventRecordTo`, 창을 올리지 않고 포커스만 주는 도구들이 쓰는 경로). 레코드는 0xf8 바이트 버퍼에 길이@0x04, `CGSEventType`@0x08, `location`@0x10, `windowLocation`@0x20, 창 ID@0x3c로 배치되고, 다운(1)·업(2)·드래그(6) 타입에 실제 좌표를 써 넣으면 전면 앱을 바꾸지 않고도 위치 지정 탭·드래그가 착지한다. 모든 쓰기는 버퍼 안쪽이므로 잘못된 값은 크래시가 아니라 무동작이 된다.

키보드는 아직 이 경로가 없다. 이벤트 레코드 레이아웃이 구현되지 않아 `type_text`와 `press`는 잠깐 창을 활성화하는 기본 경로로 되돌아간다. 즉 **마우스 동작은 완전 백그라운드, 키보드는 아니다**라는 비대칭이 남아 있다.

방해는 두 종류로 분리해 회계한다. `focus.probe`로 전후 스냅샷을 뜨고 `focus.diff`가 `raised`와 `stole_focus`를 돌려준다. 창이 사용자가 보던 것을 **덮었는지**(raised)와 키보드 포커스만 가져갔는지(stole_focus)는 다른 실패이고, 전자만 실질적 방해다.

### F. 창 정체성: 지역화된 이름을 쓰지 않는다

미러링 창을 찾는 코드에는 값비싸게 배운 세부가 남아 있다. 창은 소유자 **PID**로 매칭하고, 번들 ID `com.apple.ScreenContinuity`의 실행 프로세스에서 찾는다. 소유자 이름(`kCGWindowOwnerName`)은 지역화되므로, 중국어·번체 중국어 시스템에서는 앱 이름이 그 언어의 문자열로 보고되어 영어 상수와의 동등 비교가 영원히 실패하고 "폰이 연결되지 않음"만 반복된다. 번들 ID는 지역화되지 않는다.

크기 필터도 제거됐다. Stage Manager가 비활성 창을 왼쪽 레일에 약 38×130으로 주차하면, 그것은 화면에 있고 실제로 폰인 창인데 예전의 `Width < 100` 가드가 이를 버려서 다른 앱이 무대를 가져갈 때마다 연결이 끊긴 것으로 보고됐다. PID와 레이어 0 조건만으로 패널·툴바는 이미 배제된다.

### G. 클라우드: 어휘를 HTTPS로 옮기기

폰이 없거나 한 대 더 필요할 때, 같은 27개 연산이 HTTP로 전달된다. 각 연산은 `<control url>/op`에 `{"op", "kw"}` POST 한 번이 되고, 응답이 그 연산의 값이 된다. 스크린샷은 PNG 바이트로 오고 로컬 백엔드가 썼을 경로에 그대로 쓰이므로, 상위 헬퍼는 폰이 어디 있는지 알지 못한다. 능력 목록은 `GET <control url>/ops`이고, `supports(op)`가 그것을 읽는다. 호스트가 목록에 없는 연산을 보내면 POST 이전에 `Unsupported`로 거부된다.

세션 모델은 두 종류다. **임시 Android**는 세션마다 새 기기이고, **저장된 Android**는 앱과 로그인을 유지한 채 되살아난다. 세션은 기본 15분, 최대 30분이며 **연장이 불가능**하다. 클라우드 iPhone은 고정 구독이라 분당 과금이 없고, 잠들거나 잠금 화면·대기 화면(StandBy)에 있으면 `wake()`가 깨워서 입력 가능 상태로 만든다(암호는 절대 입력하지 않으며, 이 폰들은 암호 없이 설정된다).

---

## 3. 정량 벤치마크 (Quantitative Evidence)

이 저장소는 모델 벤치마크나 과제 성공률을 공개하지 않는다. 따라서 정량 근거는 **구현에 박힌 측정값과 기기별 능력 행렬, 그리고 서비스 계약**이다. 추측으로 채우지 않고 확인 가능한 것만 적는다.

### A. 기기별 관측·조작 능력 행렬

| 기기 | 도달 경로 | 눈 | 손 | 텍스트 정확도 |
| :--- | :--- | :--- | :--- | :--- |
| 개인 iPhone | macOS 미러링 창 + 사용자 잠금 | 스크린샷 + Vision OCR | HID CGEvents(또는 SkyLight 레코드) | OCR 추론(`pixels`) |
| 데스크톱 Android | adb(USB 또는 Wi-Fi) | 접근성 트리(정확), Mac에서는 Vision 폴백 | adb `input` | 트리(`tree`) |
| 클라우드 Android | adb(호스트로) | 접근성 트리, Mac에서 Vision 폴백 | adb `input` | 트리(`tree`) |
| 클라우드 iPhone | HTTPS | 서버 측 OCR | HTTPS 연산 | OCR 추론(`pixels`) |

이 행렬이 곧 이 프로젝트의 비용 구조다. iPhone 경로만 유일하게 "아이콘은 보이지 않고 라벨만 보인다"는 제약을 가지며, 그래서 아이콘 전용 컨트롤은 스크린샷과 비전 모델이 필요하다.

### B. 구현에 고정된 시간 상수

| 상수 | 값 | 이유 |
| :--- | :--- | :--- |
| `nav.home` 후 대기 | 0.8초 | 미러링이 홈 전환을 그릴 시간 |
| Spotlight 실행 | `Cmd+3`, 0.9초 → 입력, 1.2초 → Enter | 결과 목록이 채워지기 전 커밋 방지 |
| 스크롤 후 1차 대기 | 0.4초, 이후 0.25초 간격 폴링 | 정착 판정의 최소 간격 |
| `scroll_screen` 정착 예산 | 2.5초 | 지연 로딩 스피너가 풀릴 시간 |
| `scroll_collect` 정체 허용 | 3회 연속 무신규 | 느린 지연 로딩에 대한 여유 |
| `wait_stable` | `settle=2`, 최대 6초, 0.5초 간격 | MD5 동일 연속 횟수 |
| `wait_for_text` | 10초, 0.5초 간격 | 폴링 기본값 |
| `wait_for_app` | 10초, 0.3초 간격 | Android에서는 약 0.1초 검사 |
| 트리 유휴 거부 기억 | 20초 | uiautomator 자체 대기가 약 10초 |
| `uiautomator dump` 타임아웃 | 30초 | 잘린 트리·전이 실패 재시도 경계 |
| 클라우드 세션 | 기본 15분, 최대 30분 | 연장 불가, 2분 이하에서 경고 |

### C. 클라우드 계약과 비용

| 항목 | 값 |
| :--- | :--- |
| 임시/저장 Android | 분당 과금(세션 유지 중), 정지 시 저장 Android는 디스크 상태 보존 |
| 클라우드 iPhone | 고정 구독, 분당 과금 없음, 세션 간 데이터 유지 |
| iPhone 예약 | 대당 20달러 환불 가능 보증금, 대기열 순번은 결제 순서, 월/연 플랜, 플랜 시작 후 보증금은 환불 불가 |
| 예약 한도 | 계정당 시간당 미결제 예약 5건, 취소해도 자리는 비지 않음 |
| 파일 전송 | 세션과 함께 15분 내 만료되는 서명 링크, 세션 종료 시 파일 삭제 |

### D. 저장소 규모

`src/phone_harness`는 약 8,700줄이고, 그중 클라우드 클라이언트가 2,493줄, Android 백엔드가 1,097줄, 미러 백엔드가 807줄, iOS 백엔드가 756줄이다. 에이전트용 `SKILL.md`가 628줄로, 코드보다 지침이 얇지 않다. 이 비율 자체가 설계의 성격을 보여준다. **기계적 조작은 작고, 판단 규율이 크다.**

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **OCR은 라벨만 본다.** 아이콘, 이미지, 상태 표시(체크 표시·회색 처리)는 문자열이 아니므로 보이지 않는다. 라벨 없는 컨트롤은 스크린샷과 비전 가능 모델이 필요하고, 이것은 결정론적 파이프라인이 아니라 확률적 판단이다.
2. **트리가 없는 곳에서는 신뢰도가 곧 오류율이다.** Vision은 깨끗한 텍스트에 1.0을 주므로 신뢰도가 높다고 정확한 것이 아니다. `min_confidence=0.3` 기본값은 노이즈를 줄이는 타협이지 보장이 아니다.
3. **멀티터치·카메라·Face ID 흐름이 없다.** 핀치 줌, 두 손가락 스크롤, 얼굴 인증이 필요한 결제·잠금 해제 경로는 다룰 수 없다. DRM 영상은 검게 렌더링된다.
4. **잠금이 미러링을 멈춘다.** iPhone을 잠그면 미러링이 일시정지되고, PIN이 걸린 Android는 사용자가 풀어야 한다. 연결은 언제나 사용자의 일이다.
5. **키보드 입력이 포커스를 되찾는다.** 마우스 동작은 백그라운드로 처리되지만 `type_text`/`press`는 창을 잠깐 활성화한다. 즉 "완전히 방해 없는 자동화"는 타이핑이 들어가는 순간 깨진다.
6. **비공개 API에 의존한다.** SkyLight의 `SLPSPostEventRecordTo` 심볼은 macOS 빌드마다 보장되지 않는다. 구현은 실패 시 기본 미러 경로로 폴백하지만, 그러면 포커스 탈취가 돌아온다.
7. **제스처 의미가 OS 버전에 종속된다.** macOS 26에서 세로 터치 드래그가 미러링에 의해 드롭되어, 리스트나 피드에서 `swipe("up"/"down")`은 아무것도 움직이지 않는다. 그래서 스크롤은 휠 이벤트(`input.scroll`)로 보내고, `swipe`는 홈 화면 페이지 전환처럼 수평 동작에만 남았다. 두 동사가 서로 반대 방향 규약을 쓰는 것도 이 때문이다(콘텐츠 방향 대 손가락 방향).
8. **정착 판정은 휴리스틱이다.** 시계·애니메이션·라이브 액티비티가 계속 바뀌면 화면은 영원히 정착하지 않고, 그때는 예산을 다 쓴 뒤의 관측을 믿어야 한다. 반대로 캐시된 화면이 두 번 동일하게 읽히면 아직 로딩 중인데도 정착으로 판정될 수 있다.
9. **세션은 연장되지 않는다.** 클라우드 Android 세션이 만료되면 그 명령은 실패하고, 하네스는 **다른 폰으로 대신 넘어가지 않는다.** 저장된 Android는 데이터를 보존하지만 실행 상태는 잃는다. 클라우드 iPhone도 마찬가지로, 죽은 세션의 메모를 보고한 뒤 같은 폰을 더 긴 시간으로 다시 시작해야 한다.
10. **링크는 자격증명이다.** 파일 전송 링크와 `url.open`에 쓰이는 링크는 15분 내 만료되지만 그동안은 인증 수단이다. 구현 주석이 출력·노트에 붙여넣지 말라고 명시하는 이유다. `adb` 주소는 비밀이 아니지만 잠금 해제 코드는 비밀이다.
11. **되돌릴 수 없는 행동의 표면이 넓다.** 이것은 사용자의 실제 폰과 실제 계정이다. 전송·게시·팔로우·구매·삭제·설정 변경은 승인 대상이고, PIN·비밀번호·2FA 코드는 절대 입력하지 않는다. 클라우드 폰에서는 통제권을 사용자에게 넘기는(`cloud open`) 경로가 유일한 안전한 인계다.
12. **권한 요구가 크다.** macOS 쪽은 실행 프로세스에 손쉬운 사용(Accessibility)과 화면 기록(Screen Recording) 권한을 요구한다. 그 권한은 폰뿐 아니라 그 Mac의 모든 창을 읽고 조작할 수 있는 권한이다.
13. **성숙도.** 패키지 분류가 `Development Status :: 3 - Alpha`이고 버전은 0.3.3이다. 2026-08 생성 후 3개월여에 5,820 스타와 615 포크가 붙은 빠른 프로젝트로, 문서와 코드가 함께 움직인다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **화면을 API로 다루는 계약 설계.** 관측 결과에 `source`를 붙이는 작은 결정은, "정확한 문자열"과 "추론된 문자열"을 같은 타입으로 섞지 않겠다는 선언이다. GUI 자동화·컴퓨터 사용 에이전트 전반에서 재사용할 수 있는 최소한의 인식론적 표지다.
- **어휘 이전(vocabulary portability)이 추상화의 단위다.** 프로토콜이 없는 두 세계(SkyLight 이벤트 레코드와 adb)를 하나로 묶은 것은 공통 와이어 포맷이 아니라 **의미 이름 27개**였다. 새 기기를 추가하는 비용이 "27개 중 몇 개가 가능한가"로 정량화되고, 불가능은 예외가 아니라 부재로 표현된다.
- **에뮬레이션 거부.** 없는 기능을 그럴듯하게 흉내 내지 않는 정책은 자동화 신뢰성의 핵심이다. 가짜 뒤로가기는 호출자가 구분할 수 없는 오답을 만들고, 그 오답은 상위 계획 전체를 오염시킨다.
- **행동–기대 술어 루프.** 반환값이 없는 환경에서 성공을 정의하는 일반 패턴이다. "무엇이 바뀌어야 하는지 먼저 이름 붙인다"는 규칙은 GUI 에이전트 평가 하네스에서 성공 판정 기준으로 바로 옮길 수 있다.
- **끝 판정을 움직임에 두기.** 파서 기반 종료 조건은 밀집 리스트에서 조기 종료를 만든다. 화면 이동 기반 종료는 앱이 리스트를 평행 이동시키든 피드처럼 항목을 교체하든 동작한다. 스크롤 수집 벤치마크를 설계할 때 재사용 가능한 규약이다.
- **방해 회계(interruption accounting).** `raised`와 `stole_focus`를 분리해 기록하는 것은, 사람과 화면을 공유하는 자동화의 품질 지표가 될 수 있다. 지연·정확도만 재는 GUI 에이전트 평가에 "사용자가 얼마나 방해받았는가"라는 축을 추가한다.
- **에이전트 작업 공간의 자기 확장.** 헬퍼는 코어에만 있지 않다. 에이전트는 `~/.phone-harness/agent-workspace/agent_helpers.py`에 자기 헬퍼를 쌓고, 다음 실행에서 그것이 전역으로 로드된다. 능력 축적이 모델 가중치 밖에서 일어나는 구조이며, 업그레이드가 이 폴더를 덮어쓰지 않는다.
- **후속 연구 방향**: (i) OCR 신뢰도 보정과 캘리브레이션된 폴백 정책, (ii) 아이콘 그라운딩(라벨 없는 컨트롤의 결정론적 위치화), (iii) 키보드 이벤트 레코드 구현으로 완전 백그라운드 타이핑, (iv) 정착 판정의 적응적 예산(화면 복잡도 추정 기반), (v) 장기 실행 과제에서의 실패 복구 정책 학습, (vi) 접근성 트리와 픽셀 관측의 융합(동일 요소를 두 출처로 교차 검증).

### 참고 링크

- 저장소: https://github.com/phone-harness/phone-harness
- 에이전트 지침(작업 규율·기기별 절): https://github.com/phone-harness/phone-harness/blob/main/SKILL.md
- 설치·문제 해결: https://github.com/phone-harness/phone-harness/blob/main/install.md , 온보딩: https://github.com/phone-harness/phone-harness/blob/main/onboarding.md
- 연산 어휘 정의: https://github.com/phone-harness/phone-harness/blob/main/src/phone_harness/transport.py
- 백그라운드 입력(SkyLight 레코드): https://github.com/phone-harness/phone-harness/blob/main/src/phone_harness/background.py
- iOS 백엔드·미러 전송: https://github.com/phone-harness/phone-harness/blob/main/src/phone_harness/ios.py , https://github.com/phone-harness/phone-harness/blob/main/src/phone_harness/mirror.py
- Android 백엔드: https://github.com/phone-harness/phone-harness/blob/main/src/phone_harness/android.py
- 클라우드 클라이언트: https://github.com/phone-harness/phone-harness/blob/main/src/phone_harness/cloud.py , HTTPS 백엔드: https://github.com/phone-harness/phone-harness/blob/main/src/phone_harness/remote.py
- 문서·대시보드: https://phone-harness.com/docs
