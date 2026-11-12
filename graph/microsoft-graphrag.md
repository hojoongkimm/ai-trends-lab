# 🕸️ Microsoft GraphRAG: A Modular Graph-based Retrieval-Augmented Generation System

- **Repository**: [microsoft/graphrag](https://github.com/microsoft/graphrag)
- **Stars**: ⭐ 36,000+
- **Category**: `Graph` / `Graph ML & LLM Fusion`
- **Key Themes**: Knowledge Graph Extraction, Community Detection, Hierarchical Summarization, Global Search

---

## 1. 개요 (Overview)
`GraphRAG`는 마이크로소프트 리서치(Microsoft Research)에서 발표한 혁신적인 지식 그래프 기반 RAG 파이프라인입니다. 기존의 단순 벡터 임베딩 기반 검색(Naive RAG)이 복잡한 데이터셋 전체에 대한 포괄적 질문(Global Sensemaking, 예: "이 문서 전체의 핵심 테마는 무엇인가?")에 취약한 점을 지식 그래프와 커뮤니티 탐지(Leiden 알고리즘)로 극복했습니다.

## 2. 핵심 아키텍처 및 메커니즘 (Key Mechanisms)

1. **Entity & Relationship Extraction (엔티티/관계 추출)**:
   - 비정형 텍스트 덩어리에서 LLM을 이용해 노드(Entity), 엣지(Relationship), 클레임(Claim)을 자동 추출하여 지식 그래프 구축.

2. **Hierarchical Community Detection (계층적 커뮤니티 탐지)**:
   - 그래프 클러스터링 알고리즘(Leiden)을 적용해 노드들을 상위/하위 커뮤니티로 계층적 그룹화.

3. **Pre-computed Community Summaries (커뮤니티 요약 사전 생성)**:
   - 각 군집별로 LLM이 서술형 요약 리포트를 생성하여 보관함으로써, 질의 시점에 수천 개의 문서를 매번 스캔할 필요 없이 고수준 요약문만으로 전체 데이터셋을 아우르는 글로벌 질의 응답 실현.

## 3. 연구 및 실무 활용 포인트 (Research & Practical Impact)
- **GNN/그래프 연구**: 텍스트 데이터로부터 대규모 도메인 지식 그래프를 구성하는 전처리 파이프라인으로 표준 활용.
- **연구실 과제**: 방대한 학술 논문, 특허, 기술 문서 집합에서 크로스 도메인 인사이트를 자율 도출하는 RAG 고도화에 즉시 적용 가능.
