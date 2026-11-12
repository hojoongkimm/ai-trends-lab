# ⚡ learn-claude-code: A Nano Claude Code-like Agent Harness

- **Repository**: [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)
- **Stars**: ⭐ 80,000+
- **Category**: `Harness` / `Educational & Lightweight Architecture`
- **Key Themes**: Bash is all you need, 0 to 1 Implementation, Agent Loop, CLI Tool Execution

---

## 1. 개요 (Overview)
`learn-claude-code`는 앤트로픽의 최신 CLI 에이전트 도구인 `Claude Code`의 내부 동작 메커니즘을 순수 Bash와 경량 스크립트만으로 바닥부터 구현한 오픈소스 교육 및 실험용 하네스입니다. 무거운 프레임워크(LangChain 등) 없이 에이전트의 핵심 루프가 어떻게 작동하는지 투명하게 보여줍니다.

## 2. 핵심 아키텍처 및 메커니즘 (How It Works)

1. **Pure Bash / POSIX Execution**:
   - 의존성 라이브러리 없이 표준 셸 환경에서 실행 가능.
   - 셸 파이프라인(`grep`, `sed`, `awk`, `curl`)을 활용한 초고속 입출력 파싱.

2. **ReAct (Reasoning + Acting) Agent Loop**:
   - `LLM API 호출` ➔ `JSON/도구 명령 파싱` ➔ `로컬 셸/도구 실행` ➔ `실행 결과 수집(STDOUT/STDERR)` ➔ `다음 턴 반영`으로 이어지는 무한 에이전트 제어 루프를 간결한 코드로 규격화.

3. **Compact Context Compaction**:
   - 턴이 길어질 때 이전 툴 실행 결과를 요약하여 토큰 한도를 방어하는 메모리 압축 로직 포함.

## 3. 평가 및 시사점 (Takeaways)
- 에이전트 하네스의 본질은 복잡한 추상화가 아니라 **'신뢰할 수 있는 도구 실행 환경과 정밀한 시스템 프롬프트 피드백 루프'**에 있음을 실증함.
- 경량 서버나 엣지 디바이스에서 독립형 AI 코딩 비서를 직접 구동할 때 최고의 베이스라인 코드.
