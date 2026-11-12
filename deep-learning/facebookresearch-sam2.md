# 👁️ Segment Anything Model 2 (SAM 2): Visual Foundation Model for Images and Videos

- **Repository**: [facebookresearch/sam2](https://github.com/facebookresearch/sam2)
- **Stars**: ⭐ 20,000+ (SAM 1은 55,000+)
- **Category**: `Deep Learning` / `Computer Vision & Video Foundation Model`
- **Organization**: Meta FAIR
- **Key Themes**: Real-time Video Segmentation, Promptable Visual Segmentation, Memory Attention

---

## 1. 개요 (Overview)
Meta FAIR에서 공개한 `SAM 2`는 이미지뿐만 아니라 비디오 전체 프레임에서 모든 객체를 실시간으로 분할(Segmentation)하고 추적할 수 있는 범용 비주얼 파운데이션 모델입니다.

## 2. 핵심 아키텍처 (Architecture)
1. **Streaming Memory Architecture**:
   - 과거 프레임의 객체 특징(Memories)을 메모리 뱅크에 축적하고, 현재 프레임에서 Memory Attention 메커니즘을 통해 객체가 가려지거나 형태가 변해도 끊김 없이 추적.
2. **Interactive Prompting**:
   - 클릭 한 번, 바운딩 박스, 또는 대략적인 마스크 입력만으로 복잡한 비디오 시퀀스 전체에 분할 마스크를 실시간 전파.
3. **High Efficiency**:
   - 경량화된 인코더/디코더 설계로 단일 GPU에서 40fps 이상의 실시간 비디오 처리 속도 달성.

## 3. 연구 적용점 (Research Impact)
- 비전 기반 결함 검사, 자율주행 객체 추적, 의료 영상 비디오 분석 등 컴퓨터 비전 분야의 강력한 전처리 및 베이스라인 모델로 필수 활용.
