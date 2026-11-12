# 📊 Cleanlab: The Standard for Data-Centric AI & Label Error Detection

- **Repository**: [cleanlab/cleanlab](https://github.com/cleanlab/cleanlab)
- **Stars**: ⭐ 10,000+
- **Category**: `Machine Learning` / `Data-Centric AI`
- **Key Themes**: Confident Learning, Noisy Labels, Outlier Detection, Dataset Quality Auditing

---

## 1. 개요 (Overview)
`Cleanlab`은 MIT 연구진이 발표한 'Confident Learning' 이론을 기반으로, 머신러닝 데이터셋 내의 잘못된 라벨(Label Issues), 이상치(Outliers), 중복 데이터를 자동으로 찾아내고 보정해 주는 데이터 중심(Data-Centric) AI 표준 라이브러리입니다.

## 2. 주요 기능 (Features)
- **Confident Learning 기반 라벨 노이즈 정제**:
  - 모델의 예측 확률 분포와 불확실성을 모델링하여 어떤 데이터 포인트의 라벨이 잘못 기재되었는지 수학적으로 증명 및 감지.
- **Model-Agnostic**:
  - PyTorch, Scikit-learn, XGBoost, Hugging Face 등 모든 ML/DL 모델의 출력 확률값과 즉시 호환.
- **실제 벤치마크 검증**:
  - ImageNet, MNIST, CIFAR 등 전 세계 주요 벤치마크 데이터셋에서 수천 개의 잘못 라벨링된 오류를 실제로 찾아내며 신뢰성 입증.
