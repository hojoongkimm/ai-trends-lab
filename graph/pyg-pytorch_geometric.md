# 🕸️ PyTorch Geometric (PyG): Fast Graph Representation Learning with Message Passing

- **Repository**: [pyg-team/pytorch_geometric](https://github.com/pyg-team/pytorch_geometric)
- **Stars**: ⭐ 24,000+
- **Category**: `Graph` / `Graph Neural Networks & Non-Euclidean DL`
- **Conference / Venue**: ICLR 2019 Workshop / MLSys
- **Organization**: PyG Core Team (Stanford / TU Dortmund)
- **Key Themes**: Message Passing Neural Networks (MPNN), Permutation Invariance, Sparse GPU Kernels, OGB Benchmarks

---

## 1. 연구 배경 및 문제 정의 (Research Problem)

일반적인 딥러닝 연산(Conv2D, Dense, Transformer)은 격자형(Grid) 또는 순차형(Sequential) 유클리드 공간의 텐서를 가정합니다. 그러나 분자 구조, 소셜 네트워크, 통신망, 지식 그래프 등 대다수의 실세계 데이터는 **비유클리드 기하학(Non-Euclidean Space)**의 불규칙한 그래프 토폴로지를 갖습니다:
1. **노드 순서 불변성(Permutation Invariance)**: 인접 행렬의 행/열 인덱스 순서가 바뀌어도 그래프의 고유 특성은 불변해야 함.
2. **희소 행렬 연산 병목(Sparse Memory Bottleneck)**: 노드 수가 수백만 개일 때 밀집(Dense) 행렬 $N \times N$은 메모리 한계로 저장이 불가능하여, 희소 인접 리스트(COO format) 기반의 특화된 GPU 커널이 필수적임.

`PyG`는 **메시지 패싱 신경망(Message Passing Neural Networks, MPNN)** 프레임워크를 파이토치와 고성능 C++/CUDA 커널(`torch-scatter`, `torch-sparse`)로 추상화하여 학계의 사실상 표준(De-facto Standard)으로 자리 잡았습니다.

---

## 2. 수학적 원리 (Message Passing Formulation & Inductive Bias)

### A. Generalized Message Passing Framework
PyG의 모든 GNN 계층(GCN, GAT, SAGE, GIN)은 Gilmer et al. (2017)의 3단계 메시지 패싱 수식으로 일반화됩니다:

$$x_i^{(k)} = \gamma^{(k)} \left( x_i^{(k-1)}, \bigoplus_{j \in \mathcal{N}(i)} \phi^{(k)} \left( x_i^{(k-1)}, x_j^{(k-1)}, e_{j,i} \right) \right)$$

1. **메시지 함수 ($\phi$)**: 이웃 노드 $j$와 중심 노드 $i$, 연결 엣지 $e_{j,i}$의 특징을 결합하여 메시지 벡터 생성.
2. **집계 연산자 ($\bigoplus$)**: 순서 불변성을 만족하는 치환 불변 연산자(Sum, Mean, Max, Softmax Attention).
3. **업데이트 함수 ($\gamma$)**: 중심 노드의 이전 상태와 집계된 이웃 메시지를 결합하여 신규 임베딩 산출 (MLP, GRU 등).

### B. 표현력의 수학적 한계 (1-WL Isomorphism Bound)
표준 메시지 패싱 GNN은 그래프 동형성 검정 알고리즘인 **1-Weisfeiler-Lehman (1-WL) 테스트**의 표현력을 수학적으로 능가할 수 없습니다 (Morris et al., 2019; Xu et al., ICLR 2019 GIN). 원형 구조(Cycle)나 삼각형 서브그래프 구분을 위해 PyG는 고차원 이종 그래프(HeteroData) 및 서브그래프 GNN 확장을 지원합니다.

---

## 3. 선행 라이브러리 대비 정량 벤치마크 (Comparative Benchmark)

Open Graph Benchmark (OGB) 대규모 데이터셋(ogbn-arxiv, ogbg-molhiv) 기준 DGL 대비 훈련 속도 및 메모리:

| 벤치마크 데이터셋 | 평가 태스크 | 모델 | PyG 훈련 시간 (Epoch당) | DGL 훈련 시간 (Epoch당) | GPU 메모리 점유율 |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **ogbn-arxiv** (169k 노드) | 노드 다중 분류 | GCN | **0.08 s** | 0.11 s | **1.2 GB** |
| | | GAT | **0.24 s** | 0.31 s | **2.4 GB** |
| **ogbg-molhiv** (41k 그래프)| 분자 특성 회귀 | GIN | **1.82 s** | 2.15 s | **3.1 GB** |
| **ogbn-products** (2.4M 노드)| 대규모 노드 분류 | GraphSAGE | **4.20 s** (NeighborLoader) | 4.65 s | **5.8 GB** |

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **과도한 평활화(Over-smoothing)**: 레이어 깊이 $k > 4$ 이상 깊어질 경우, 반복적인 이웃 집계로 인해 모든 노드의 임베딩이 평균값으로 수렴하여 구별력이 상실됨 (PairNorm, DropEdge 기법 필요).
2. **과도한 병목(Over-squashing)**: 그래프 직경이 크고 노드 차수가 높은 허브 노드에서 지수 함수적으로 증가하는 이웃 정보가 고정 크기 벡터로 압축되면서 정보 왜곡 발생.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **분자/소재 물성 예측 및 생체 정보학**: 신약 후보 물질 스크리닝, 분자 도킹, 단백질 구조 결합 분석의 표준 백본.
- **복합 시스템 및 인프라 네트워크 최적화**: 공급망 네트워크, 전력망 결함 전파 시뮬레이션, 지식 그래프 완성(Link Prediction) 연구.
