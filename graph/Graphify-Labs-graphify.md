# 🕸️ Graphify: Turn Any Codebase and Docs into a Queryable Knowledge Graph

- **Repository**: [Graphify-Labs/graphify](https://github.com/Graphify-Labs/graphify)
- **Stars**: ⭐ 146,000+
- **Category**: `Graph` / `Codebase Intelligence & Knowledge Graph`
- **Key Themes**: Local Deterministic AST, Knowledge Graph, No Vector Store, Code Analysis, Skill for Coding Agents

---

## 1. 개요 (Overview)
`Graphify`는 대규모 소프트웨어 코드베이스, 문서, SQL 스키마, 설정 파일 등을 파싱하여 로컬에서 즉시 쿼리 가능한 결정론적(Deterministic) 지식 그래프로 변환해 주는 초고속 지식 그래프 추출 도구입니다.

## 2. 핵심 메커니즘 (Key Mechanisms)
- **Zero Vector Store**: 부정확한 유사도 기반 벡터 검색에 의존하지 않고, 코드의 AST(추상 구문 트리)와 심볼 의존성(Call Graph, Type Dependency)을 100% 명시적 그래프 엣지로 연결.
- **Explainable Edges**: 모든 엣지가 "왜 연결되었는지"에 대한 결정론적 근거를 보유하여 에이전트의 코드 파악 환각 방지.
- **Agent Skill Integration**: Claude Code, Cursor, Codex 등 코딩 에이전트에 `/graphify` 스킬 형태로 통합되어 파일 수천 개 규모의 프로젝트도 단번에 파악 가능.
