# ⚡ ECC: The Agent Harness Performance Optimization System

- **Repository**: [affaan-m/ECC](https://github.com/affaan-m/ECC)
- **Stars**: ⭐ 300,000+
- **Category**: `Harness` / `Agent Architecture`
- **Key Themes**: Claude Code, Codex, Opencode, Cursor, Performance Optimization, Memory, Agent Security

---

## 1. 개요 (Overview)
`ECC`는 Claude Code, Codex, Cursor 등 최근 급부상한 코딩 및 작업 실행 AI 에이전트들의 런타임 하네스(Agent Harness) 성능을 극대화하기 위해 설계된 시스템 최적화 프레임워크입니다. 에이전트의 단순 텍스트 생성을 넘어, 파일시스템 접근, 셸 실행, 메모리 보존, 도구 보안 경계를 엔지니어링 수준에서 통제합니다.

## 2. 핵심 아키텍처 및 특징 (Key Features)

1. **Instincts & Memory Layer (본능 및 적응형 메모리)**:
   - 세션 간 지식 유실(Context drop)을 방지하기 위해 계층형 메모리 구조 채택.
   - 단기 작업 컨텍스트(Scratchpad)와 장기 프로필/프로젝트 규칙(Instinct)을 분리하여 컨텍스트 윈도우 오버헤드를 40% 이상 절감.

2. **Dynamic Skill Registry (동적 스킬 레지스트리)**:
   - 모든 도구를 사전에 프롬프트에 주입하는 대신, 작업 의도에 따라 필요한 스킬(Skill) 명세를 온디맨드로 로드.
   - 토큰 비용을 최소화하고 모델의 도구 선택 정확도(Tool Selection Accuracy) 향상.

3. **Execution Sandbox & Security Guardrails**:
   - 무제한 셸 권한 부여로 인한 부작용을 막기 위한 명령어 화이트리스트 및 타임아웃/리소스 쿼터 관리.
   - 위험 명령어 실행 전 자율 검증 계층 내장.

## 3. 연구 및 실무 활용 포인트 (Research & Practical Impact)
- **대학원 연구**: 자율 에이전트의 다단계 계획(Planning) 및 실행(Execution) 평가 벤치마크 테스트 환경으로 활용 가능.
- **연구실 과제**: 대규모 코드베이스 리팩토링이나 반복 실험 스크립트 실행 시 에이전트 하네스로 도입하여 안정적인 무인 실행 파이프라인 구축 가능.
