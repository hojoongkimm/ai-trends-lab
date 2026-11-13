# 👁️ vLLM: Efficient Memory Management for Large Language Model Serving with PagedAttention

- **Repository**: [vllm-project/vllm](https://github.com/vllm-project/vllm)
- **Stars**: ⭐ 47,000+
- **Category**: `Deep Learning` / `Serving Systems & Inference Optimization`
- **Conference / Venue**: ACM SOSP 2023 (Symposium on Operating Systems Principles)
- **Organization**: UC Berkeley LMSYS
- **Key Themes**: PagedAttention, Continuous Batching, KV-Cache Fragmentation Reduction, Speculative Decoding

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

LLM 추론(Inference) 단계는 자기회귀(Autoregressive) 생성 특성상 **메모리 대역폭 집약적(Memory-bound)**입니다. 특히 이미 계산된 키-값 텐서를 보관하는 **KV 캐시(KV Cache)** 관리가 GPU 서빙의 최대 병목입니다:
1. **극심한 메모리 단편화(Fragmentation)**: 기존 서빙 시스템(FasterTransformer, HuggingFace)은 요청 시 최대 시퀀스 길이(예: 2,048 토큰)를 미리 가정하고 연속적인 물리 메모리를 선할당(Pre-allocation)하므로, 실제 토큰이 생성되기 전까지 내부 단편화(Internal Fragmentation)와 외부 단편화로 인해 **GPU 메모리의 60%~80%가 낭비**되었습니다.
2. **배치 크기의 인위적 제약**: KV 캐시의 불필요한 선할당으로 인해 동시 처리 가능한 배치 크기(Batch Size)가 극단적으로 제한되어 GPU 연산기(Tensor Core) 활용률이 바닥에 머물렀습니다.

`vLLM`은 운영체제(OS)의 가상 메모리 페이징 기법을 신경망 어텐션에 최초로 도입한 **PagedAttention**을 고안하여, KV 캐시 낭비를 4% 미만으로 억제하고 추론 처리량을 최대 4배 향상시켰습니다.

---

## 2. 수학적 원리 및 아키텍처 (Mathematical Principles & Systems Architecture)

### A. PagedAttention Formulation
기존 어텐션은 연속적인 메모리 블록 $K, V \in \mathbb{R}^{L \times d}$를 필요로 합니다:
$$A = \text{Softmax}\left(\frac{Q K^\top}{\sqrt{d}}\right) V$$

PagedAttention은 KV 캐시를 고정 크기 블록(Block Size $B=16$ 토큰) 단위로 비연속적인 물리 메모리 페이지(Physical Blocks)에 분할 저장합니다:

$$K = [K_{(1)}, K_{(2)}, \dots, K_{(m)}], \quad V = [V_{(1)}, V_{(2)}, \dots, V_{(m)}]$$

어텐션 연산 시, 블록 테이블(Block Table)을 통해 논리적 인덱스를 물리 메모리 포인터로 매핑하며 커널 레벨에서 블록별 내적을 분할 누적(Block-wise Online Softmax)합니다:

$$A_{ij} = \frac{\exp(q_i k_j^\top / \sqrt{d})}{\sum_{l} \exp(q_i k_l^\top / \sqrt{d})}$$

이로 인해 메모리 할당이 필요할 때마다 1개 블록(16 토큰)씩 온디맨드로 동적 할당되므로 내부 단편화가 사실상 $0$에 수렴합니다.

### B. Iteration-Level Continuous Batching
전통적인 요청 단위 정적 배칭(Static Batching)과 달리, 토큰 생성 1회 반복(Step)마다 완료된 요청은 즉시 퇴출(Evict)하고 대기 큐의 새 요청을 즉시 슬롯에 채워 넣는 반복 수준 스케줄링(Iteration-level Scheduling)을 수행합니다.

---

## 3. 선행 서빙 엔진 대비 정량 벤치마크 (Comparative Benchmark)

NVIDIA A100-80GB GPU, LLaMA-70B 모델 추론 서빙 처리량(Throughput, Requests/sec):

| 서빙 프레임워크 | 메모리 낭비율 (Fragmentation) | 서빙 처리량 (Req/s) | 지연시간 (P99 Latency) | VLM(멀티모달) 지원 |
| :--- | :---: | :---: | :---: | :---: |
| **Hugging Face TGI (초기)** | ~73% | 1.8 req/s | 1,420 ms | ⚠️ 제한적 |
| **FasterTransformer** | ~62% | 2.4 req/s | 1,180 ms | ❌ 불가 |
| **TensorRT-LLM (초기)** | ~55% | 3.1 req/s | 950 ms | ⚠️ 복잡한 변환 |
| **vLLM (PagedAttention, Ours)** | **< 3.8%** | **7.6 req/s (4x)** | **820 ms** | **✅ Native VLM 지원** |

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **블록 테이블 룩업 오버헤드**: 시퀀스 길이가 짧은 단순 단문 질의에서는 블록 테이블 간접 참조(Indirect Memory Access)로 인한 커널 오버헤드가 연속 메모리 어텐션 대비 미세하게 발생할 수 있음.
2. **동적 텐서 병렬화(Dynamic TP) 전환 제약**: 런타임 중에 GPU 개수를 유연하게 변경하기 어려우며 초기 클러스터 토폴로지 설정에 종속됨.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **대규모 추론 파이프라인의 인프라 표준**: 연구실 GPU 서버에서 오픈소스 LLM(Llama, Qwen, DeepSeek) 및 멀티모달 모델을 단일/멀티 GPU 엔드포인트로 호스팅할 때 필수 기반 엔진.
- **Speculative Decoding 및 프리픽스 캐싱 결합**: 연구용 고속 추론 및 에이전트 다단계 루프 가속에 최적화된 백엔드.
