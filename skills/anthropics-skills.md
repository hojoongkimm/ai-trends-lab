# 🛠️ anthropics/skills: Anthropic Official Public Agent Skills Repository

- **Repository**: [anthropics/skills](https://github.com/anthropics/skills)
- **Stars**: ⭐ 193,000+
- **Category**: `Skills` / `Official Specification`
- **Key Themes**: Anthropic Official, Tool Calling, Computer Use, Skill Design Standards

---

## 1. 개요 (Overview)
`anthropics/skills`는 앤트로픽(Anthropic)이 공식 공개한 에이전트 스킬 프로토콜 및 표준 구현 저장소입니다. Claude의 도구 사용(Tool Use)과 컴퓨터 사용(Computer Use) 기능을 표준화된 인터페이스로 패키징하는 글로벌 벤치마크 역할을 합니다.

## 2. 규격 및 설계 철학 (Design Principles)
1. **Clear Boundaries (명확한 책임 분리)**: 각 스킬은 독립된 환경에서 입력 파라미터 검증, 에러 처리, 결과 직렬화를 완결하도록 설계.
2. **Context-Aware Tool Descriptions**: 모델이 환각 없이 도구를 호출할 수 있도록 명세서 작성 가이드라인 제시.
3. **Enterprise Security Standards**: 임의 코드 실행 및 파일시스템 접근 시의 보안 샌드박스 정책 정의.

