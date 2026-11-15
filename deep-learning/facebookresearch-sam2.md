# 👁️ Segment Anything Model 2 (SAM 2): Real-Time Visual Foundation Model for Images and Videos

- **Repository**: [facebookresearch/sam2](https://github.com/facebookresearch/sam2)
- **Stars**: ⭐ 20,000+
- **Category**: `Deep Learning` / `Computer Vision & Video Foundation Models`
- **Conference / Venue**: Meta FAIR Technical Report / arXiv 2024
- **Organization**: Meta FAIR
- **Key Themes**: Promptable Visual Segmentation (PVS), Streaming Memory Attention, Memory Bank, Occlusion Handling

---

## 1. 연구 배경 및 문제 정의 (Research Problem)

2023년 발표된 SAM 1은 단일 이미지 분할(Image Segmentation)에서 강력한 제로샷(Zero-shot) 일반화 능력을 입증했으나, 비디오 영역으로 확장할 때 다음과 같은 구조적 한계에 부딪혔습니다:
1. **시간적 일관성(Temporal Consistency) 부재**: 프레임 단위로 독립 추론 시 프레임 간 객체 아이덴티티가 깜빡거리거나(Flickering) 가려짐(Occlusion) 후 추적이 유실됨.
2. **비디오 메모리 폭발**: 기존 비디오 객체 분할(VOS) 모델들은 전체 비디오 프레임 간의 양방향 3D 어텐션을 수행하여 메모리 복잡도가 $O(T^2)$로 증가, 장시간 비디오 처리가 불가능함.

`SAM 2`는 <strong>스트리밍 메모리 아키텍처(Streaming Memory Architecture)</strong>와 **메모리 어텐션 메커니즘**을 도입하여, 단일 이미지와 실시간 비디오 스트림을 단일 통합 아키텍처로 통일했습니다.

---

## 2. 수학적 원리 및 아키텍처 (Mathematical Principles & Architecture)

### A. Streaming Memory Attention & Memory Bank
현재 프레임 $t$의 비전 특징 맵 $F_t$가 주어졌을 때, 과거 프레임들의 정보는 **Memory Bank** $\mathcal{M}_t$에 FIFO 큐 형태로 보관됩니다 (과거 프레임 임베딩 및 사용자의 대화형 프롬프트 임베딩).

$$\mathcal{M}_t = \lbrace m_1, \dots, m_{t-1}, p_1, \dots, p_k \rbrace$$

현재 프레임의 특징은 Memory Cross-Attention을 통해 업데이트됩니다:

$$\tilde{F}_t = \text{Softmax}\left( \frac{Q(F_t) K(\mathcal{M}_t)^\top}{\sqrt{d}} + \mathbf{P}_{\text{rel}} \right) V(\mathcal{M}_t)$$

여기서 $\mathbf{P}_{\text{rel}}$은 시간적 거리(Temporal distance)를 인코딩하는 상대적 시간 위치 임베딩(Relative Positional Embedding)입니다. 이 구조 덕분에 메모리 복잡도는 전체 비디오 길이 $T$에 무관하게 고정된 메모리 뱅크 크기 $M$에 비례하여 $O(M \cdot B)$로 엄격히 제한됩니다.

### B. Occlusion Head & Ambiguity Resolution
객체가 프레임 밖으로 벗어나거나 완전히 가려진 상태를 판별하기 위해 전용 **Occlusion Prediction Head**를 두어 마스크의 존재 확률 $p_{\text{vis}} \in [0, 1]$을 동시에 예측합니다.

---

## 3. 선행 모델 대비 정량 벤치마크 (Comparative Benchmark)

DAVIS-2017, MOSE, SA-V 벤치마크 기준 비디오 객체 분할(VOS) 정량 성능:

| 모델 아키텍처 | DAVIS-2017 $\mathcal{J}$ & $\mathcal{F}$ | MOSE $\mathcal{J}$ & $\mathcal{F}$ (어려운 가림 환경) | 추론 속도 (FPS, RTX 4090) | 비디오-이미지 통합 지원 |
| :--- | :---: | :---: | :---: | :---: |
| **SAM 1 (Frame-by-frame)** | 54.2 | 31.8 | 12 fps | ❌ 이미지 전용 |
| **XMem (ECCV 2022)** | 86.2 | 67.5 | 22 fps | ❌ 인터랙티브 프롬프트 한계 |
| **DeAOT (NeurIPS 2022)** | 86.8 | 69.1 | 26 fps | ❌ 비디오 전용 |
| **Cutie (CVPR 2024)** | 88.5 | 73.4 | 28 fps | ❌ 이미지 분할 일반화 한계 |
| **SAM 2 (FAIR, Ours)** | **90.8** | **78.6** | **44 fps** | **✅ 이미지 & 비디오 완전 통합** |

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **극심한 텍스처 유사성 및 군집 객체 혼동**: 물고기 떼, 새 떼, 유사한 색상의 축구 유니폼 선수들이 밀집하여 교차할 때 객체 ID 전이(ID Switching) 현상 발생.
2. **장시간 누적 오차(Drift over extreme long sequences)**: 수천 프레임 이상 단 한 번의 사용자 프롬프트 교정 없이 진행될 경우, 점진적인 마스크 경계 확장(Drift)이 발생할 수 있음.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **산업용 실시간 비전 결함 추적**: 연속 생산 라인 및 컨베이어 벨트 비디오 스트림에서 미세 스크래치나 이물질의 이동 경로를 프레임 레이트 저하 없이 추적.
- **멀티모달 비디오 파운데이션 모델의 시각적 토크나이저**: 비디오 LLM(Video-LLM) 연구에서 프레임별 시공간 객체 토큰(Spatial-temporal Object Tokens)을 추출하는 표준 인코더로 채택.

