# 🕸️ LightRAG: Simple and Fast Retrieval-Augmented Generation with Dual-Level Graphs

- **Repository**: [HKUDS/LightRAG](https://github.com/HKUDS/LightRAG)
- **Stars**: ⭐ 40,000+
- **Category**: `Graph` / `Graph ML & Efficient RAG`
- **Conference**: EMNLP 2025
- **Key Themes**: Dual-level Graph Indexing, Low-cost Graph RAG, Fast Entity Retrieval

---

## 1. 개요 (Overview)
홍콩대학교 데이터인텔리전스 연구소(HKU Data Intelligence Lab)에서 개발한 `LightRAG`는 Microsoft GraphRAG의 높은 인덱싱 비용과 느린 속도를 획기적으로 개선한 2계층(Dual-level) 그래프 기반 RAG 프레임워크입니다.

## 2. 핵심 아키텍처 및 차별점 (Core Architecture)

1. **Dual-Level Graph Indexing (2단계 그래프 인덱싱)**:
   - **Low-level**: 구체적인 엔티티 및 그들 사이의 세밀한 엣지 관계 캡처.
   - **High-level**: 상위 테마 및 추상적 개념 노드 연결.
   - 단일 질의에 대해 미시적 세부 정보와 거시적 맥락을 동시에 탐색.

2. **Graph-Enhanced Vector Retrieval**:
   - 노드 임베딩과 그래프 토폴로지 정보를 함께 활용하여 검색 속도를 기존 GraphRAG 대비 수 배 이상 단축하고 토큰 비용을 대폭 절감.

3. **Incremental Update Support**:
   - 새로운 문서 추가 시 전체 그래프를 재구축할 필요 없이 증분 노드/엣지만 동적 삽입 가능.
