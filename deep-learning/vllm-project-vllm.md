# 👁️ vLLM: High-Throughput and Memory-Efficient LLM & VLM Serving Engine

- **Repository**: [vllm-project/vllm](https://github.com/vllm-project/vllm)
- **Stars**: ⭐ 47,000+
- **Category**: `Deep Learning` / `Serving Engine & Systems`
- **Key Themes**: PagedAttention, Continuous Batching, High Throughput, Multimodal VLM Support

---

## 1. 개요 (Overview)
`vLLM`은 UC 버클리에서 시작되어 AI 서빙 인프라의 표준으로 자리잡은 초고속 LLM 및 비전-언어모델(VLM) 추론 엔진입니다. 전통적인 서빙 시스템 대비 최대 2~4배 높은 처리량(Throughput)을 제공합니다.

## 2. 핵심 기술 혁신 (Key Innovations)
- **PagedAttention (페이징 어텐션)**:
   - OS의 가상 메모리 페이징 기법에서 영감을 받아, KV 캐시 메모리 단편화(Fragmentation)를 사실상 0% 수준으로 억제.
- **Continuous Batching (지속적 배치 처리)**:
   - 요청마다 생성 길이가 다른 특성을 반영하여, 시퀀스 단위가 아닌 토큰 반복 단위로 동적 배칭을 수행하여 GPU 연산 낭비 제거.
- **Multimodal Native**:
   - LLaVA, Qwen-VL, InternVL 등 최신 멀티모달 비전 모델의 엔드포인트 서빙 완벽 지원.
