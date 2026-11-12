# ⚡ LM-Evaluation-Harness: The Standard Few-Shot LLM Evaluation Framework

- **Repository**: [EleutherAI/lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness)
- **Stars**: ⭐ 14,500+
- **Category**: `Harness` / `Evaluation & Benchmarking`
- **Key Themes**: LLM Benchmark, Few-shot Evaluation, MMLU, GSM8k, Hugging Face Leaderboard Standard

---

## 1. 개요 (Overview)
`lm-evaluation-harness`는 EleutherAI에서 주도하고 글로벌 AI 연구 생태계에서 사실상의 표준(De-facto Standard)으로 채택한 오픈소스 언어모델 벤치마크 평가 프레임워크입니다. Hugging Face의 Open LLM Leaderboard의 백엔드 평가 엔진으로 사용됩니다.

## 2. 주요 기능 및 구성 (Features)
1. **60개 이상의 표준 벤치마크 지원**: MMLU, GSM8k, ARC, HellaSwag, TruthfulQA, HumanEval 등 주요 평가 태스크 내장.
2. **다양한 추론 백엔드 호환**: vLLM, Hugging Face Transformers, DeepSpeed, SGLang, Megatron 등과 즉시 연동.
3. **재현성(Reproducibility) 보장**: 엄격한 프롬프트 포맷팅, 토큰 확률 계산, Zero-shot / Few-shot 템플릿 표준화.

## 3. 연구 활용 가치 (Research Value)
- 자체 개발한 파인튜닝 모델이나 가중치 양자화 모델의 성능 저하 여부를 객관적인 정량 지표로 검증할 때 필수적인 도구.
