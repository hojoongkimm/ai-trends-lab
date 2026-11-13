# 📊 TabM: Advancing Tabular Deep Learning With Parameter-Efficient Ensembling

- **Repository**: [yandex-research/tabm](https://github.com/yandex-research/tabm)
- **Stars**: ⭐ 1,100+
- **Category**: `Machine Learning` / `Tabular Deep Learning`
- **Conference / Venue**: ICLR 2025
- **Organization**: Yandex Research
- **Key Themes**: Parameter-Efficient Ensembling (PEE), Tabular DL vs. GBDT, BatchEnsemble Adaptation, Variance Reduction

---

## 1. 연구 배경 및 문제 정의 (Research Problem)

정형 데이터(Tabular Data) 영역은 오랫동안 트리 기반 앙상블 모델(Gradient Boosted Decision Trees: XGBoost, LightGBM, CatBoost)이 지배해 왔습니다. 심층 신경망(MLP, ResNet, FT-Transformer)은 연속형 피처의 회전 불변성(Rotational Invariance) 가정과 고르지 못한 손실 지형(Loss Landscape)으로 인해 다음과 같은 한계를 보였습니다:
1. **과적합 및 높은 일반화 오차**: 정형 데이터는 피처 간 스케일 차이가 크고 비선형 상호작용이 불규칙하여, 단일 딥러닝 모델의 가중치가 특정 노이즈 패턴에 쉽게 과적합됨.
2. **앙상블 연산 비용의 폭발**: 성능 격차를 줄이기 위해 통상 10~30개의 독립 신경망 앙상블을 수행하지만, 훈련/추론 시간과 메모리 사용량이 $O(M)$배로 선형 증가하여 실무 배포가 불가능함.

`TabM`은 **단일 모델의 계산 비용($<10\%$ 오버헤드)으로 $M$개의 서브 모델 앙상블 효과**를 내는 파라미터 효율적 앙상블(Parameter-Efficient Ensembling) 아키텍처를 제시하여, 정형 데이터 딥러닝이 GBDT를 체계적으로 능가할 수 있음을 증명했습니다.

---

## 2. 수학적 원리 및 아키텍처 (Mathematical Formulation)

### A. Rank-1 Perturbation Parameterization
기존 앙상블이 $M$개의 독립적인 가중치 행렬 $\{W_m\}_{m=1}^M \in \mathbb{R}^{d_{\text{out}} \times d_{\text{in}}}$을 학습하는 반면, TabM은 공유 가중치 행렬 $W_0$에 앙상블 멤버별 Rank-1 스케일링 벡터 $(r_m, s_m)$를 하다마르 곱(Hadamard product) 형태로 결합합니다:

$$W_m = W_0 \odot (r_m s_m^\top), \quad r_m \in \mathbb{R}^{d_{\text{out}}}, \; s_m \in \mathbb{R}^{d_{\text{in}}}$$

순전파(Forward Pass) 연산 시, 미니배치 내 입력 $x$에 대해 랭크-1 벡터를 브로드캐스팅하여 효율적으로 병렬 계산합니다:
$$h_m = (W_0 (x \odot s_m)) \odot r_m + b_m$$

### B. 파라미터 및 계산 복잡도 비교

| 기법 | 파라미터 수 | 순전파 FLOPs | 메모리 복잡도 |
| :--- | :---: | :---: | :---: |
| **Naive Deep Ensemble ($M$개)** | $O(M \cdot d_{\text{in}} d_{\text{out}})$ | $O(M \cdot B \cdot d_{\text{in}} d_{\text{out}})$ | $O(M \cdot d_{\text{in}} d_{\text{out}})$ |
| **TabNet / FT-Transformer** | $O(L \cdot d^2)$ | $O(B \cdot L \cdot d^2)$ | $O(B \cdot L \cdot d)$ |
| **TabM ($M$ 서브 앙상블)** | **$O(d_{\text{in}} d_{\text{out}} + M(d_{\text{in}} + d_{\text{out}}))$** | **$O(B \cdot d_{\text{in}} d_{\text{out}} + M \cdot B(d_{\text{in}} + d_{\text{out}}))$** | **$O(d_{\text{in}} d_{\text{out}} + M(d_{\text{in}} + d_{\text{out}}))$** |

파라미터 증가율이 $M \cdot (d_{\text{in}} + d_{\text{out}}) / (d_{\text{in}} d_{\text{out}}) \approx 0.05$ 미만으로 억제되어, 연산 오버헤드가 단일 MLP 대비 $5\sim 10\%$에 불과합니다.

---

## 3. 선행 모델 대비 벤치마크 및 비교 분석 (Comparative Analysis)

45개 공개 벤치마크 데이터셋(OpenML Tabular Benchmark) 기준 정량 비교:

| 모델 분류 | 대표 아키텍처 | 튜닝 전 랭킹 (Avg Rank) | 앙상블 튜닝 후 랭킹 | 단점 및 병목 |
| :--- | :--- | :---: | :---: | :--- |
| **Tree Ensembles** | XGBoost / LightGBM | 2.8 | 2.1 | 비정형 데이터(텍스트/이미지)와의 멀티모달 결합 불가 |
| | CatBoost | 2.2 | 1.8 | 고차원 희소(Sparse) 데이터에서 메모리 폭발 |
| **Traditional DL** | MLP | 5.4 | 4.6 | 회전 불변성으로 인한 과적합, 축 정렬 결정경계 학습 취약 |
| | ResNet-Tabular | 4.1 | 3.5 | 얕은 최적화 수렴 한계 |
| **Transformer DL**| FT-Transformer | 3.2 | 2.7 | $O(P^2)$ 셀프 어텐션 연산으로 대규모 피처에서 극도로 느림 |
| **Proposed** | **TabM (Mini-Ensemble)** | **1.6** | **1.2** | **단일 훈련 비용으로 GBDT 최상위 성능 능가** |

---

## 4. 한계점 및 실패 모드 (Failure Modes & Trade-offs)

1. **극단적 희소 범주형 피처(High-Cardinality Sparse Features)**: 카테고리 종류가 수십만 개인 경우 임베딩 테이블 크기가 커져 Rank-1 스케일링 벡터의 정규화 효과가 반감됨.
2. **비정규화 이상치(Unnormalized Outliers)**: GBDT는 값의 순서(Rank)에만 의존하므로 이상치에 완전히 강건하지만, TabM은 입력 스케일 정규화(Quantile / Robust Scaling)에 민감함.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **정형 딥러닝 베이스라인의 세대교체**: 정형 데이터를 다루는 학술 논문에서 단순 MLP나 ResNet 대신 현대적 SOTA 베이스라인으로 필수 채택 필요.
- **도메인 특화 결합 가능성**: 물리 법칙 기반 신경망(PINN)이나 센서 데이터 분석에 TabM 블록을 백본으로 결합하여 단일 모델의 분산(Variance)을 억제하는 고정밀 서로게이트 모델 구축 가능.

