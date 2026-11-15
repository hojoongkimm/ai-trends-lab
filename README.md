# 🚀 AI Trend Archive

[![Tracked Categories](https://img.shields.io/badge/Tracked%20Categories-8-blue.svg)](#-categories)
[![Analysis Reports](https://img.shields.io/badge/Analysis%20Reports-24-blue.svg)](#-categories)
[![Auto Curation](https://img.shields.io/badge/Curation-Daily%20Updated-brightgreen.svg)](#-curation--tracking-policy)
[![Open Source](https://img.shields.io/badge/Open%20Source-Awesome%20List-orange.svg)](#)

> **Curated archive of trending, novel, and high-impact AI, Machine Learning, Deep Learning, Graph Neural Networks, LLM, Agent, Harness, Automation, and Skill repositories.**  
> 인공지능(AI) 전 분야—그래프(GNN), 딥러닝(비전/멀티모달), 머신러닝(정형/최적화/데이터중심), 에이전트, 하네스, 스킬, LLM, 자동화—에서 스타가 급상승하거나 패러다임을 혁신하는 오픈소스 프로젝트를 체계적으로 분석하여 아카이빙합니다.

---

## 📂 Categories & Quick Navigation

| 카테고리 | 설명 | 주요 키워드 | 문서 수 |
| :--- | :--- | :--- | :---: |
| [**🕸️ Graph**](./graph) | 그래프 신경망(GNN), 지식 그래프, Graph RAG | Graph Neural Networks, PyG, GraphRAG, Knowledge Graph | 4 |
| [**👁️ Deep Learning**](./deep-learning) | 비전/비디오 파운데이션 모델, 서빙 인프라 | SAM 2, vLLM, Visual Foundation, Serving Systems | 2 |
| [**📊 Machine Learning**](./machine-learning) | 정형 딥러닝, 데이터 중심 AI, 라벨 품질 정제 | TabM, Cleanlab, Tabular DL, Confident Learning | 2 |
| [**🤖 Agent**](./agent) | 자율 AI 에이전트, 멀티에이전트 스웜, 인지 루프 | Autonomous Agent, Swarms, Cognitive Memory | 3 |
| [**⚡ Harness**](./harness) | 에이전트 실행 환경, 벤치마크 평가, 런타임 제어 | Agent Harness, Eval Harness, Execution Boundary | 5 |
| [**🛠️ Skills**](./skills) | 에이전트 도구 사용, 컴퓨터 인터랙션, 모듈형 역량 | Agent Skills, Tool-Use, Computer Use | 5 |
| [**⚙️ Automation**](./automation) | 경량 에이전트 자동화, 워크플로우 오케스트레이션 | Headless Agent, VPS Deployment, Scalable Loops | 2 |
| [**🧠 LLM**](./llm) | 파운데이션 모델 응용, Agentic RAG, 구조화 추론 | LLM Apps, Agentic RAG, Production Reasoning | 1 |

---

## 🌟 Featured Trends (Key Tracked Repositories)

### 1. Graph & Graph Neural Networks (GNN)
- [**Graphify (`Graphify-Labs/graphify`)**](./graph/Graphify-Labs-graphify.md) ⭐ `146k+`  
  *Codebase to Knowledge Graph* — 대규모 코드와 문서를 AST 기반 결정론적 지식 그래프로 변환해 에이전트에 공급하는 혁신적 그래프 도구.
- [**LightRAG (`HKUDS/LightRAG`)**](./graph/HKUDS-LightRAG.md) ⭐ `40k+` (EMNLP 2025)  
  *Dual-Level Graph RAG* — 2단계 계층적 그래프 인덱싱을 통해 저비용·초고속으로 포괄적 질의를 해결하는 최신 그래프 RAG.
- [**Microsoft GraphRAG (`microsoft/graphrag`)**](./graph/microsoft-graphrag.md) ⭐ `36k+`  
  *Hierarchical Community Graph RAG* — Leiden 커뮤니티 탐지와 LLM 요약을 결합한 마이크로소프트의 글로벌 지식 그래프 검색 시스템.
- [**PyTorch Geometric (`pyg-team/pytorch_geometric`)**](./graph/pyg-pytorch_geometric.md) ⭐ `24k+`  
  *Premier GNN Library* — GCN, GAT, GraphSAGE 및 OGB 벤치마크를 완벽 지원하는 딥러닝 그래프 라이브러리의 표준.

### 2. Deep Learning & Foundation Models
- [**Segment Anything Model 2 (`facebookresearch/sam2`)**](./deep-learning/facebookresearch-sam2.md) ⭐ `20k+`  
  *Visual Foundation for Images & Videos* — Meta FAIR의 실시간 비디오 및 이미지 객체 분할/추적 비주얼 파운데이션 모델.
- [**vLLM (`vllm-project/vllm`)**](./deep-learning/vllm-project-vllm.md) ⭐ `47k+`  
  *High-Throughput Serving Engine* — PagedAttention과 지속적 배치 처리로 GPU 메모리 낭비를 없앤 고성능 추론 엔진.

### 3. Machine Learning (Tabular & Data-Centric)
- [**TabM (`yandex-research/tabm`)**](./machine-learning/yandex-research-tabm.md) ⭐ `1.1k+` (ICLR 2025)  
  *Parameter-Efficient Tabular Deep Learning* — 단일 모델 훈련 비용으로 강력한 앙상블 효과를 내며 GBDT에 필적하는 정형 딥러닝 SOTA.
- [**Cleanlab (`cleanlab/cleanlab`)**](./machine-learning/cleanlab-cleanlab.md) ⭐ `10k+`  
  *Data-Centric AI Standard* — Confident Learning 이론 기반으로 데이터셋 내 라벨 오류와 이상치를 자동 탐지/보정하는 필수 도구.

### 4. Harness & Execution Runtime
- [**ECC (`affaan-m/ECC`)**](./harness/affaan-m-ECC.md) ⭐ `300k+`  
  *Agent Harness Optimization* — Claude Code, Codex, Cursor를 위한 스킬·본능(Instinct)·장기 기억·보안 경계 최적화 프레임워크.
- [**learn-claude-code (`shareAI-lab/learn-claude-code`)**](./harness/shareAI-lab-learn-claude-code.md) ⭐ `80k+`  
  *Nano Agent Harness* — 0부터 1까지 순수 Bash로 Claude Code 아키텍처를 구현한 경량 에이전트 하네스.
- [**Open Code Review (`alibaba/open-code-review`)**](./harness/alibaba-open-code-review.md) ⭐ `50k+`  
  *Deterministic × Agent Hybrid* — 파일 선별·룰 매칭을 결정론적으로 고정하고 에이전트는 동적 판단에만 쓰는 하이브리드 코드 리뷰 하네스로, AACR-Bench에서 정밀도 우선 트레이드오프를 정량 제시.
- [**LM-Evaluation-Harness (`EleutherAI/lm-evaluation-harness`)**](./harness/EleutherAI-lm-evaluation-harness.md) ⭐ `14.5k+`  
  *Few-shot LLM Benchmark Standard* — 글로벌 빅테크 및 연구소 전반에서 표준으로 사용하는 언어모델 벤치마크 평가 하네스.
- [**Plannotator (`backnotprop/plannotator`)**](./harness/backnotprop-plannotator.md) ⭐ `11.1k+`  
  *Human Decision Surface* — 계획·코드 리뷰를 호스트 훅에 매달아 승인 시 편집본을 실행 입력으로 되돌리고, nonce·개정·스냅샷의 3중 토큰으로 화면의 아티팩트만 승인되게 하는 20호스트 어댑터 하네스.

### 5. Skills & Tool Ecosystem
- [**Anthropic Agent Skills (`anthropics/skills`)**](./skills/anthropics-skills.md) ⭐ `193k+`  
  *Official Agent Skills* — 도구 호출 및 컴퓨터 제어를 규격화한 앤트로픽 공식 에이전트 스킬 프로토콜.
- [**Superpowers (`obra/superpowers`)**](./skills/obra-superpowers.md) ⭐ `316k+`  
  *Agentic Skills Framework* — TDD 기반 소프트웨어 엔지니어링 규율을 에이전트에 주입하는 스킬 프레임워크.
- [**Agent Skills (`addyosmani/agent-skills`)**](./skills/addyosmani-agent-skills.md) ⭐ `116k+`  
  *Production-Grade Skills* — 구글 엔지니어링 리드 Addy Osmani의 프로덕션 엔지니어링 스킬 팩.
- [**Modern Web Guidance (`GoogleChrome/modern-web-guidance-src`)**](./skills/GoogleChrome-modern-web-guidance-src.md) ⭐ `4.7k+`  
  *Eval-Calibrated Skill Corpus* — 학습 데이터의 레거시 편향을 겨냥해 250개 웹 기능·203개 사용 사례를 오프라인 하이브리드 검색으로 주입하고, 평가 상승분 기반 프루닝으로 토큰을 절감하는 스킬 코퍼스.
- [**Chrome DevTools MCP (`ChromeDevTools/chrome-devtools-mcp`)**](./skills/ChromeDevTools-chrome-devtools-mcp.md) ⭐ `56.3k+`  
  *Browser Observation Substrate* — 접근성 트리 스냅샷과 `(loaderId, backendNodeId)` 기반 안정 핸들, DevTools TraceEngine을 그대로 쓰는 성능 통찰 패리티, 63개 도구와 토큰 예산 산출 설계.

### 6. Agent Systems & Swarms
- [**ruflo (`ruvnet/ruflo`)**](./agent/ruvnet-ruflo.md) ⭐ `81k+`  
  *Multi-Player Swarms* — 적응형 메모리, 벡터 RAG, 멀티에이전트 조율을 결합한 자율 에이전트 플랫폼.
- [**CowAgent (`zhayujie/CowAgent`)**](./agent/zhayujie-CowAgent.md) ⭐ `47k+`  
  *Personal Assistant Harness* — 태스크 플래닝, 스킬 실행, 자체 지식 진화를 지원하는 경량 멀티채널 에이전트.
- [**LoopForge (`Tencent/LoopForge`)**](./agent/Tencent-LoopForge.md) ⭐ `900+`  
  *Durable SOP Delivery* — 실행마다 워크플로 그래프를 동결하고 임차·세대 펜싱으로 소유권을 보장하며, 사람 게이트를 검증되는 판정 아티팩트로 승격한 10개 CLI 오케스트레이션 런타임.

### 7. Automation & Scalability
- [**OpenHuman (`tinyhumansai/openhuman`)**](./automation/tinyhumansai-openhuman.md) ⭐ `42k+`  
  *High-Density Harness* — $10 저가 VPS 위에서 500개 이상의 백그라운드 에이전트를 동시 구동하는 초경량 하네스.
- [**Codex Security (`openai/codex-security`)**](./automation/openai-codex-security.md) ⭐ `13k+`  
  *Multi-Pass Vulnerability Pipeline* — 표준/딥 스캔 중복제거와 결함 정체성 스키마, 탐지-검증-패치-재검증 루프를 갖춘 에이전트 보안 감사 자동화.

### 8. LLM & Applications
- [**Awesome LLM Apps (`Shubhamsaboo/awesome-llm-apps`)**](./llm/Shubhamsaboo-awesome-llm-apps.md) ⭐ `144k+`  
  *100+ Production Apps* — 최신 멀티모달, 에이전틱 검색, 지능형 RAG 아키텍처 실무 코드베이스 모음.

---

## 📌 Curation & Tracking Policy

1. **탐색 범위 (Coverage)**:
   - **그래프 (Graph/GNN)**: 지식 그래프, Graph RAG, OGB 벤치마크, 이종 그래프 신경망
   - **딥러닝 (Deep Learning)**: 컴퓨터 비전/비디오 파운데이션 모델, 디퓨전, 추론/서빙 가속 엔진
   - **머신러닝 (Machine Learning)**: 정형 데이터 딥러닝, AutoML, 데이터 중심 AI, 라벨 노이즈 정제
   - **에이전트/하네스/스킬 (Agentic AI)**: 자율 인지 루프, 런타임 보안/하네스, 휴먼 인 더 루프 승인·리뷰 표면, 브라우저 관측·도구 사용 표면, 엔지니어링 스킬 팩
   - **LLM/자동화 (LLM & Automation)**: 에이전틱 RAG, 코드 리뷰·보안 감사 파이프라인, 경량 백그라운드 오케스트레이션
2. **분석 기준**:
   - 아키텍처 및 핵심 구조 요약
   - 기존 모델/도구 대비 차별점 및 벤치마크 SOTA 성과
   - 최신 AI 연구 및 엔지니어링 프로젝트 적용 방안
3. **업데이트 주기**: 글로벌 오픈소스 생태계 트렌드 기반 지속적 큐레이션 및 업데이트.

