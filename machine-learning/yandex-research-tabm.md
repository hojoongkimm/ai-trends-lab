# 📊 TabM: Advancing Tabular Deep Learning With Parameter-Efficient Ensembling

- **Repository**: [yandex-research/tabm](https://github.com/yandex-research/tabm)
- **Stars**: ⭐ 1,100+
- **Category**: `Machine Learning` / `Tabular Deep Learning`
- **Conference**: ICLR 2025
- **Organization**: Yandex Research
- **Key Themes**: Tabular DL vs GBDT, Parameter-Efficient Ensembling, Mini-Ensemble Layer

---

## 1. 개요 (Overview)
`TabM`은 오랫동안 트리 기반 모델(XGBoost, LightGBM, CatBoost)이 지배해 온 정형 데이터(Tabular Data) 영역에서, 딥러닝 모델이 파라미터 효율적 앙상블(PEE)을 통해 GBDT 수준의 강력한 성능과 앙상블 효과를 단일 모델 훈련 비용으로 달성할 수 있음을 증명한 ICLR 2025 화제 논문 구현체입니다.

## 2. 핵심 메커니즘 (Key Mechanisms)
1. **BatchEnsemble-style Efficient Weight Multiplying**:
   - 가중치 행렬 전체를 여러 개 복제하는 대신, 단일 가중치에 입력/출력 단위의 랭크-1 스케일링 벡터 세트만 추가하여 k개의 서브 모델 앙상블 효과 생성.
2. **Computational Overhead < 10%**:
   - 단일 신경망 모델을 돌리는 것과 거의 동일한 FLOPs 및 훈련 시간으로 수십 개의 앙상블 모델을 결합한 것과 동등한 정규화 및 일반화 성능 확보.
3. **Tabular Deep Learning의 새로운 SOTA**:
   - 40개 이상의 공개 벤치마크 데이터셋에서 기존 MLP, ResNet, FT-Transformer, SAINT 등을 큰 격차로 압도.

## 3. 산업공학/데이터사이언스 연구 활용 가치 (Research Value)
- 산업 현장의 센서 로그, 제조 공정 데이터, 설비 고장 예측 등 정형 시계열/테이블 데이터에 딥러닝을 적용할 때 가장 강력하고 가벼운 최신 아키텍처.
