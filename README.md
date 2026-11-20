# 🚀 AI Trend Archive

[![Tracked Categories](https://img.shields.io/badge/Tracked%20Categories-8-blue.svg)](#-categories--quick-navigation)
[![Analysis Reports](https://img.shields.io/badge/Analysis%20Reports-42-blue.svg)](#-categories--quick-navigation)
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
| [**🤖 Agent**](./agent) | 자율 AI 에이전트, 멀티에이전트 스웜, 지식 공급 계층 | Autonomous Agent, Swarms, Retrieval Infrastructure | 7 |
| [**⚡ Harness**](./harness) | 에이전트 실행 환경, 벤치마크 평가, 런타임 제어 | Agent Harness, Eval Harness, Execution Boundary | 9 |
| [**🛠️ Skills**](./skills) | 에이전트 도구 사용, 컴퓨터 인터랙션, 모듈형 역량 | Agent Skills, Tool-Use, Computer Use | 13 |
| [**⚙️ Automation**](./automation) | 경량 에이전트 자동화, 워크플로우 오케스트레이션 | Headless Agent, VPS Deployment, Scalable Loops | 3 |
| [**🧠 LLM**](./llm) | 파운데이션 모델 응용, 최소 재현 스택, 구조화 추론 | LLM Apps, Pretraining Reproduction, Production Reasoning | 2 |

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
- [**ECC (`affaan-m/ECC`)**](./harness/affaan-m-ECC.md) ⭐ `306k+`  
  *Harness Operating System* — 72개 에이전트·327개 스킬·98개 커맨드 위에 7단계 엔지니어링 루프를 런타임으로 얹고, 훅으로 100% 관측한 스코프 instinct를 0.3–0.9 신뢰도로 승격하며, 102개 규칙 AgentShield로 자기 구성을 감사하는 하네스 운영체제.
- [**CLI-Anything (`HKUDS/CLI-Anything`)**](./harness/HKUDS-CLI-Anything.md) ⭐ `59.7k+` (arXiv:2606.03854)  
  *Agent-Native GUI Lift* — 6튜플 계약 $H=(S,C,I,R,V,D)$와 7단계 SOP로 GUI 소프트웨어를 상태 있는 CLI로 리프팅하고, 실제 바이너리에 렌더를 위임한 4계층 검증(매직바이트·픽셀·RMS)과 미리보기 번들 프로토콜, 69개 하네스를 CLI-Hub으로 유통하는 에이전트 네이티브 인터페이스 생성기.
- [**learn-claude-code (`shareAI-lab/learn-claude-code`)**](./harness/shareAI-lab-learn-claude-code.md) ⭐ `80k+`  
  *Nano Agent Harness* — 0부터 1까지 순수 Bash로 Claude Code 아키텍처를 구현한 경량 에이전트 하네스.
- [**Open Code Review (`alibaba/open-code-review`)**](./harness/alibaba-open-code-review.md) ⭐ `50k+`  
  *Deterministic × Agent Hybrid* — 파일 선별·룰 매칭을 결정론적으로 고정하고 에이전트는 동적 판단에만 쓰는 하이브리드 코드 리뷰 하네스로, AACR-Bench에서 정밀도 우선 트레이드오프를 정량 제시.
- [**LM-Evaluation-Harness (`EleutherAI/lm-evaluation-harness`)**](./harness/EleutherAI-lm-evaluation-harness.md) ⭐ `14.5k+`  
  *Few-shot LLM Benchmark Standard* — 글로벌 빅테크 및 연구소 전반에서 표준으로 사용하는 언어모델 벤치마크 평가 하네스.
- [**Plannotator (`backnotprop/plannotator`)**](./harness/backnotprop-plannotator.md) ⭐ `11.1k+`  
  *Human Decision Surface* — 계획·코드 리뷰를 호스트 훅에 매달아 승인 시 편집본을 실행 입력으로 되돌리고, nonce·개정·스냅샷의 3중 토큰으로 화면의 아티팩트만 승인되게 하는 20호스트 어댑터 하네스.
- [**agent-browser (`vercel-labs/agent-browser`)**](./harness/vercel-labs-agent-browser.md) ⭐ `51.6k+`  
  *Shell-Native Browser Runtime* — 세션 데몬 위에서 명령 한 줄씩 브라우저를 조작하고, `(page, frame, loaderId, backendNodeId)` 정체성 기반 참조와 fail-closed 정책 로딩, 160개 MCP 도구의 프로필 양자화로 실행 경계를 계약화한 Rust 브라우저 런타임.
- [**SkillEvaluator (`NVIDIA/SkillEvaluator`)**](./harness/NVIDIA-SkillEvaluator.md) ⭐ `2.7k+`  
  *Skill Lift as a Paired Measurement* — 정적 검증·중복 검사·실측을 독립 계층으로 분리하고, ATIF 궤적 계약과 참조 미끼 기준선으로 스킬의 한계 기여를 947개 쌍 사례로 측정한 엔비디아 평가 하네스.
- [**Elephant-Goldfish Model (`google/elephant-goldfish-model`)**](./harness/google-elephant-goldfish-model.md) ⭐ `20.4k+`  
  *Review as Information Asymmetry* — 깊은 컨텍스트의 Elephant가 설계를 쓰고 제로 컨텍스트 Goldfish가 5단계 루브릭으로 검토하며, 수정 후 동일 술어를 재적용하는 회귀 단계를 둔 구글의 SDD 플러그인.

### 5. Skills & Tool Ecosystem
- [**Anthropic Agent Skills (`anthropics/skills`)**](./skills/anthropics-skills.md) ⭐ `193k+`  
  *Official Agent Skills* — 도구 호출 및 컴퓨터 제어를 규격화한 앤트로픽 공식 에이전트 스킬 프로토콜.
- [**Superpowers (`obra/superpowers`)**](./skills/obra-superpowers.md) ⭐ `321k+`  
  *Discipline as a Runtime Bootstrap* — `SessionStart` 훅으로 스킬 호출 규칙을 매 세션·압축 후 재주입하고, 스펙 합의→2–5분 과업 계획→과업당 새 서브에이전트·2단계 리뷰로 실행하며, 20,000자 압축 절단점을 CI 테스트로 고정한 15개 절차 스킬.
- [**Agent Skills (`addyosmani/agent-skills`)**](./skills/addyosmani-agent-skills.md) ⭐ `116k+`  
  *Production-Grade Skills* — 구글 엔지니어링 리드 Addy Osmani의 프로덕션 엔지니어링 스킬 팩.
- [**Modern Web Guidance (`GoogleChrome/modern-web-guidance-src`)**](./skills/GoogleChrome-modern-web-guidance-src.md) ⭐ `4.7k+`  
  *Eval-Calibrated Skill Corpus* — 학습 데이터의 레거시 편향을 겨냥해 250개 웹 기능·203개 사용 사례를 오프라인 하이브리드 검색으로 주입하고, 평가 상승분 기반 프루닝으로 토큰을 절감하는 스킬 코퍼스.
- [**Chrome DevTools MCP (`ChromeDevTools/chrome-devtools-mcp`)**](./skills/ChromeDevTools-chrome-devtools-mcp.md) ⭐ `56.3k+`  
  *Browser Observation Substrate* — 접근성 트리 스냅샷과 `(loaderId, backendNodeId)` 기반 안정 핸들, DevTools TraceEngine을 그대로 쓰는 성능 통찰 패리티, 63개 도구와 토큰 예산 산출 설계.
- [**Claude-BugHunter (`elementalsouls/Claude-BugHunter`)**](./skills/elementalsouls-Claude-BugHunter.md) ⭐ `11.4k+`  
  *Domain Skill Bundle* — 681개 공개 보고서 패턴을 24개 클래스의 83개 스킬로 색인하고, 7-Question Gate 판정과 deny-wins 범위 술어를 코드로 강제하며, 스킬 on/off ablation의 음성 기준선까지 공개한 보안 도메인 스킬 팩.
- [**Stitch Design Skills (`google-labs-code/stitch-skills`)**](./skills/google-labs-code-stitch-skills.md) ⭐ `9.2k+`  
  *Design-to-Code Skill Library* — Stitch MCP 위에서 `SITE.md`·`DESIGN.md`·토큰 파일로 관례를 외부화하고, 4단계 게이트와 AST 검사기로 일부 규칙을 기계 검증하며, 배턴 파일로 루프 상태를 컨텍스트 밖에 두는 디자인 스킬 묶음.
- [**Astryx (`meta/astryx`)**](./skills/meta-astryx.md) ⭐ `29.6k+`  
  *Design System for Two Readers* — CLI가 패키지의 문서를 직접 서빙해 사본 부패를 제거하고, `manifest --json`·`apiVersion` 봉투·`requires`/`available`로 계약을 기계가 읽게 하며, 밤마다 4개 대상 6개 차원으로 생성 코드를 채점하는 메타의 디자인 시스템.
- [**last30days (`mvanhorn/last30days-skill`)**](./skills/mvanhorn-last30days-skill.md) ⭐ `68k+`  
  *Engagement-Weighted Social Search* — 20여 소스를 출처별 품질 사전과 선형 최근성·참여 신호로 융합하고, 상태 술어를 실행 시점에 재조회해 검증하며, 수집 커버리지를 스스로 보고하는 리서치 스킬.
- [**Claude Trading Skills (`tradermonty/claude-trading-skills`)**](./skills/tradermonty-claude-trading-skills.md) ⭐ `6.3k+`  
  *Process Discipline as a Skill Graph* — 74개 스킬과 14개 워크플로 매니페스트로 절차의 순서와 산출물 의존을 타입으로 선언하고, 결측 시 닫히는 게이트와 8축 검증 행렬을 기계 판독 형태로 공개한 투자 스킬 코퍼스.
- [**ffmpeg-skill (`kajisho5/ffmpeg-skill`)**](./skills/kajisho5-ffmpeg-skill.md) ⭐ `6.8k+`  
  *Verification-Driven Media Editing* — 42개 도구의 입력 스키마를 argparse 파서에서 런타임에 파생해 CI로 드리프트를 막고, 플랫폼 적합성을 PASS/WARN/FAIL 술어로 판정하며, 하지 않은 일을 했다고 쓰면 0점을 주는 평가를 둔 로컬 미디어 스킬.
- [**diagram-design (`cathrynlavery/diagram-design`)**](./skills/cathrynlavery-diagram-design.md) ⭐ `68.6k+`  
  *Taste as an Executable Contract* — 51개 도식 타입을 타입별 참조로 분해해 컨텍스트 비용을 타입 수와 분리하고, 강조색 2개·9노드·4px 격자를 부등식으로 고정하며, 기하 검증기와 Chromium 렌더 린트·스킨 대비 게이트를 세 OS CI에서 돌리는 디자인 스킬.
- [**app-store-compliance (`mjmirza/app-store-compliance`)**](./skills/mjmirza-app-store-compliance.md) ⭐ `1.8k+`  
  *Policy as a Pre-Submit Gate* — 123개 거절 패턴을 신호·반대 신호 술어로 만들고 107개 규제 데드라인에 확인일과 원문을 붙이며, `PreToolUse` 훅으로 제출 명령을 exit 2로 막고 스캔 미완료를 CLEAR가 아닌 NOT CHECKED로 보고하는 컴플라이언스 게이트.

### 6. Agent Systems & Swarms
- [**ruflo (`ruvnet/ruflo`)**](./agent/ruvnet-ruflo.md) ⭐ `81k+`  
  *Multi-Player Swarms* — 적응형 메모리, 벡터 RAG, 멀티에이전트 조율을 결합한 자율 에이전트 플랫폼.
- [**CowAgent (`zhayujie/CowAgent`)**](./agent/zhayujie-CowAgent.md) ⭐ `47k+`  
  *Personal Assistant Harness* — 태스크 플래닝, 스킬 실행, 자체 지식 진화를 지원하는 경량 멀티채널 에이전트.
- [**LoopForge (`Tencent/LoopForge`)**](./agent/Tencent-LoopForge.md) ⭐ `900+`  
  *Durable SOP Delivery* — 실행마다 워크플로 그래프를 동결하고 임차·세대 펜싱으로 소유권을 보장하며, 사람 게이트를 검증되는 판정 아티팩트로 승격한 10개 CLI 오케스트레이션 런타임.
- [**PaperOrchestra (`google-research/paper-orchestra`)**](./agent/google-research-paper-orchestra.md) ⭐ `1.5k+`  
  *Manuscript Assembly Pipeline* — 5개 에이전트의 고정 데이터흐름, 시간 상한을 건 검색-검증 2단 문헌 탐색, 컴파일 가능성 제약과 점수 기반 롤백을 갖춘 구글 클라우드 AI 연구의 자기 벤치마크(200편 역설계) 동반 논문 작성 파이프라인.
- [**AG-UI (`ag-ui-protocol/ag-ui`)**](./agent/ag-ui-protocol-ag-ui.md) ⭐ `19k+`  
  *Typed Agent-User Event Stream* — 1.0 명세의 31개 이벤트·8개 계열, 스냅샷과 RFC 6902 델타의 이중화, 중단을 런 결과로 모델링한 재개, 알 수 없는 재료와 규격 위반을 정반대로 다루는 오류 분류를 갖춘 에이전트-애플리케이션 프로토콜.
- [**Qwen Code (`QwenLM/qwen-code`)**](./agent/QwenLM-qwen-code.md) ⭐ `30.6k+`  
  *Autonomy with Accounting* — 목표를 상태 기계로 두고 독립 검증자와 증거 타입 체계로 완료를 판정하며, 3,000만 토큰 예산과 무진전 3턴 한도, 단일 쓰기자 팀과 모델 아레나를 갖춘 오픈소스 터미널 에이전트 하네스.
- [**Firecrawl (`firecrawl/firecrawl`)**](./agent/firecrawl-firecrawl.md) ⭐ `204k+`  
  *Fallback Waterfall for Agent Knowledge* — 스크레이프·크롤·맵·검색을 15개 엔진의 우선순위 폴백으로 처리하고, 엔진별 시간 상한과 해시 기반 실험 배정을 두며, 로컬/클라우드 출처를 `source` 필드로 표기하고 미지의 설정값을 `off`로 처리하는 지식 공급 계층.

### 7. Automation & Scalability
- [**OpenHuman (`tinyhumansai/openhuman`)**](./automation/tinyhumansai-openhuman.md) ⭐ `42k+`  
  *High-Density Harness* — $10 저가 VPS 위에서 500개 이상의 백그라운드 에이전트를 동시 구동하는 초경량 하네스.
- [**Codex Security (`openai/codex-security`)**](./automation/openai-codex-security.md) ⭐ `13k+`  
  *Multi-Pass Vulnerability Pipeline* — 표준/딥 스캔 중복제거와 결함 정체성 스키마, 탐지-검증-패치-재검증 루프를 갖춘 에이전트 보안 감사 자동화.
- [**DSA (`ZhuLinsen/daily_stock_analysis`)**](./automation/ZhuLinsen-daily_stock_analysis.md) ⭐ `72.8k+`  
  *Evidence-Aware Multi-Market Research* — 데이터셋×시장 능력 행렬로 결측을 리포트에 노출하고, 위험 신호를 단조 축소 사상으로만 낮추며, 시간 초과 시 (k, n) 부분 전달을 계약으로 둔 스케줄 실행 파이프라인.

### 8. LLM & Applications
- [**Awesome LLM Apps (`Shubhamsaboo/awesome-llm-apps`)**](./llm/Shubhamsaboo-awesome-llm-apps.md) ⭐ `144k+`  
  *100+ Production Apps* — 최신 멀티모달, 에이전틱 검색, 지능형 RAG 아키텍처 실무 코드베이스 모음.
- [**MiniMind (`jingyaogong/minimind`)**](./llm/jingyaogong-minimind.md) ⭐ `66.9k+`  
  *2-Hour 64M LLM from Zero* — 설정에서 파라미터를 다시 계산해 검증하는 63.9M dense/MoE 구조(Qwen3 계열 정렬)와, 사전학습→SFT→LoRA→DPO→PPO/GRPO/CISPO→도구 호출 에이전트 RL→증류까지 단계별 시간·비용을 공개한 재현 스택.

---

## 📌 Curation & Tracking Policy

1. **탐색 범위 (Coverage)**:
   - **그래프 (Graph/GNN)**: 지식 그래프, Graph RAG, OGB 벤치마크, 이종 그래프 신경망
   - **딥러닝 (Deep Learning)**: 컴퓨터 비전/비디오 파운데이션 모델, 디퓨전, 추론/서빙 가속 엔진
   - **머신러닝 (Machine Learning)**: 정형 데이터 딥러닝, AutoML, 데이터 중심 AI, 라벨 노이즈 정제
   - **에이전트/하네스/스킬 (Agentic AI)**: 자율 인지 루프, 런타임 보안/하네스, 휴먼 인 더 루프 승인·리뷰 표면, 브라우저 관측·도구 사용 표면, 브라우저 실행 런타임과 격리 계약, 엔지니어링 스킬 팩, 도메인 지식 스킬 번들, 디자인-투-코드 스킬 파이프라인, 에이전트-애플리케이션 프로토콜, 스킬 역량 실측 평가, 자율성 예산과 완료 검증, 검증 주도 도구 표면, GUI→CLI 하네스 생성과 에이전트 네이티브 인터페이스 계약, 설정 감사와 공급망 보안
   - **LLM/자동화 (LLM & Automation)**: 에이전틱 RAG, 코드 리뷰·보안 감사 파이프라인, 경량 백그라운드 오케스트레이션, 도메인 증거 조립과 스케줄 실행
2. **분석 기준**:
   - 아키텍처 및 핵심 구조 요약
   - 기존 모델/도구 대비 차별점 및 벤치마크 SOTA 성과
   - 최신 AI 연구 및 엔지니어링 프로젝트 적용 방안
3. **업데이트 주기**: 글로벌 오픈소스 생태계 트렌드 기반 지속적 큐레이션 및 업데이트.

