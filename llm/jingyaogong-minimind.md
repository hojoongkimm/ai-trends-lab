# 🧠 MiniMind: A 64M LLM Trained from Zero in Two Hours as an Auditable Chain — Parameter Accounting, Native PyTorch Stages, and Reward Shaping for Tool-Use RL

- **Repository**: [jingyaogong/minimind](https://github.com/jingyaogong/minimind)
- **Stars**: ⭐ 66,904 (forks 8,789, created 2024-07-27, last commit 2026-11-18)
- **Category**: `LLM` / Minimal LLM Reproduction and Teaching Stack
- **Organization**: jingyaogong (individual maintainer, Apache-2.0; companion lines MiniMind-V, MiniMind-O, MiniMind-dLM, MiniMind-Linear)
- **License / Runtime**: Apache-2.0 / PyTorch only, without the high-level abstractions of `transformers`, `trl` or `peft`; one RTX 3090 reproduces the chat model; weights convert into `transformers` and export paths exist for llama.cpp, vLLM and ollama
- **Key Themes**: a 64M dense model and a 198M-A64M MoE released side by side, an architecture aligned to the Qwen3 family (pre-norm RMSNorm, SwiGLU, RoPE with YaRN extrapolation, grouped-query attention with q/k normalisation), a custom 6,400-token vocabulary, one repository that walks pretrain → SFT → LoRA → DPO → PPO/GRPO/CISPO → tool use → agentic RL → distillation, and a published hour-and-yuan cost for every stage

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

대형 언어모델을 이해하려는 사람 앞에는 두 개의 벽이 있다. 첫째는 **비용**이고, 둘째는 **추상화**다. `transformers`와 `trl`로 십여 줄을 쓰면 로딩·학습·추론이 끝나지만, 그 십여 줄 뒤에서 무엇이 일어나는지는 드러나지 않는다. 이 저장소가 겨냥하는 병목은 성능이 아니라 **재현 가능한 최소 단위**다.

1. **연산 하한(Compute Floor)**: 개인 GPU 한 장으로 전 과정을 한 번 돌릴 수 있어야 실험과 학습이 성립한다. 1B급 모델은 이 조건을 만족하지 못한다.
2. **구조 충실도(Architectural Fidelity)**: 작은 모델이라도 최신 스택(Qwen3 계열)과 같은 구조를 써야 산출물이 실제 생태계로 이전된다. 장난감 구조로 만든 모델은 배운 것을 다른 곳에 쓸 수 없다.
3. **평가의 정직성(Evaluation Honesty)**: 이 규모의 모델은 벤치마크 점수가 우연 수준 근처에 머문다. 그 숫자를 어떻게 읽어야 하는지 밝히지 않으면, 재현은 곧 과장이 된다.

응답은 세 가지다. 파라미터를 손으로 셀 수 있는 크기로 고정하고, 구조를 최신 계열에 맞추고, 단계마다 시간·비용·표준오차를 함께 공개한다. 이 저장소의 진짜 산출물은 64M 가중치가 아니라 **각 단계를 읽고 고칠 수 있는 코드와 그 비용표**다.

---

## 2. 아키텍처 및 정식화 (Architecture & Formulation)

### A. 파라미터 회계(Parameter Accounting)

구조는 decoder-only Transformer이고, 설정은 `model/model_minimind.py`의 `MiniMindConfig`가 갖는다. 기본값은 `hidden_size=768`, `num_hidden_layers=8`, `num_attention_heads=8`, `num_key_value_heads=4`, `vocab_size=6400`, `max_position_embeddings=32768`, `rope_theta=1e6`이며, FFN 중간 크기는 설정에 상수로 적혀 있지 않고 다음 식으로 유도된다.

$$
d_{ff} = 64 \cdot \left\lceil \frac{\pi d}{64} \right\rceil
$$

$d = 768$이면 $d_{ff} = 64 \cdot 38 = 2432$다. 이 값을 넣으면 총 파라미터 수를 설정만으로 재구성할 수 있다. 어텐션에서 q는 $d \times d$, k와 v는 각각 $d \times d_{kv}$($d_{kv} = 4 \cdot 96 = 384$), o는 $d \times d$이고, FFN은 gate·up·down 세 행렬이다. 블록마다 레이어 정규화 2개와 q/k 정규화 2개가 붙고, 입출력 임베딩은 묶여 있다.

$$
N = Vd + L\left(2d^{2} + 2d \cdot d_{kv} + 3d \cdot d_{ff} + 2d + 2d_{head}\right)
$$

계산하면 블록당 7,374,528개, 8개 층 58,996,224개, 묶인 임베딩 4,915,200개로 합계 **63,911,424개**, 즉 63.9M이다. 공개된 64M 표기와 일치한다. 이 일치가 중요한 이유는, 모델 카드의 숫자를 믿는 대신 구조 정의에서 다시 계산해 확인할 수 있는 저장소라는 점이다. 참고로 프로젝트는 이 크기를 GPT-3 대비 약 2,700분의 1로 설명한다.

| Model | params | len_vocab | max_pos | rope_theta | n_layers | d_model | kv_heads | q_heads | note |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| minimind-3 | 64M | 6400 | 32768 | 1e6 | 8 | 768 | 4 | 8 | Dense |
| minimind-3-moe | 198M-A64M | 6400 | 32768 | 1e6 | 8 | 768 | 4 | 8 | 4 experts / top-1 |
| minimind2-small | 26M | 6400 | 32768 | 1e6 | 8 | 512 | 2 | 8 | 이전 세대 |
| minimind2 | 104M | 6400 | 32768 | 1e6 | 16 | 768 | 2 | 8 | 이전 세대 |

### B. 구조 선택과 그 근거

블록은 pre-norm RMSNorm → GQA 어텐션 → 잔차 → RMSNorm → SwiGLU FFN → 잔차의 순서다. 어텐션은 q와 k에 각각 head 차원의 RMSNorm을 붙여(Qwen3 계열의 방식) 로짓 폭주를 억제하고, 위치 정보는 RoPE로 넣는다. `inference_rope_scaling`을 켜면 YaRN이 적용되는데, 주파수별로 선형 램프를 곱하는 형태다.

$$
f'(i) = f(i)\left(1 - \gamma_i + \frac{\gamma_i}{s}\right)
$$

$\gamma_i$는 저주파에서 0, 고주파에서 1로 가는 램프이고 $s$는 확장 배수(기본 16)다. 즉 저주파 성분은 그대로 두고 고주파만 압축해, 짧은 컨텍스트에서 학습한 위상 관계를 긴 컨텍스트로 늘린다.

MoE는 같은 블록의 FFN만 전문가 4개 + top-1 라우팅으로 바꾼다. 여기서 실무적으로 흥미로운 두 가지가 있다. 첫째, top-1일 때 게이트 가중치를 `top1 - top1.detach() + 1.0`로 계산해 순전파 값은 1, 역전파 경로는 게이트 확률로 만든다(straight-through). 둘째, 라우터 부하 균형 손실을 붙인다. 전문가 사용 비율 $p_e$와 게이트 확률 평균 $q_e$의 내적에 전문가 수를 곱한 형태다.

$$
L_{aux} = \alpha \sum_{e=1}^{E} p_e q_e, \qquad p_e = \frac{1}{T}\sum_{t=1}^{T} \mathbf{1}\left[\text{top1}(t) = e\right]
$$

$\alpha$는 5e-4다. 이 항이 없으면 초기에 몇 개 전문가가 모든 토큰을 먹고 나머지가 죽는다. 저장소는 또한 MoE의 역설을 문서에 명시한다. 추론에서는 활성 파라미터가 64M이라 빠르지만, **네이티브 PyTorch 학습에서는 토큰을 전문가별로 모아 forward를 반복하므로 같은 크기 dense보다 약 50% 느리다**. 이 솔직한 문장이 fused kernel(Triton, DeepSpeed-MoE, Megatron-LM)이 왜 필요한지를 설명한다.

### C. 단계별 비용과 재현 경로

전 과정이 스크립트 하나에 한 단계씩 대응한다. `train_tokenizer.py` → `train_pretrain.py` → `train_full_sft.py` → `train_lora.py` → `train_dpo.py` → `train_ppo.py` / `train_grpo.py` → `train_distillation.py` → `train_agent.py`. 데이터는 파일 단위로 공개되고, 최소 재현 조합은 경량 프리트레인 1.2GB와 경량 SFT 1.6GB다. 주의할 점은 "2시간·3위안"이 전체 사전학습이 아니라 **경량 프리트레인 + 경량 SFT를 3090 한 장에서 1 epoch** 돌린 값이라는 것이다. 프로젝트가 직접 그 범위를 밝힌다.

| Model | params | pretrain_t2t_mini | sft_t2t_mini | toolcall | RLAIF |
| :--- | :--- | :--- | :--- | :--- | :--- |
| minimind-3 | 64M | 약 1.21h / 1.57위안 | 약 1.10h / 1.43위안 | 약 0.9h / 1.17위안 | 약 1.1h / 1.43위안 |
| minimind-3-moe | 198M-A64M | 약 1.69h / 2.20위안 | 약 1.54h / 2.00위안 | 약 1.26h / 1.64위안 | 약 1.54h / 2.00위안 |

3090 임대 단가를 시간당 1.3위안으로 잡고, 두 단계를 합쳐 2.31시간·약 3.0위안으로 Zero 대화 모델을 얻는다. 8장 H100이면 분 단위로 줄어든다는 서술도 함께 둔다.

### D. 후처리 학습의 목적함수

선호 학습은 DPO로, 그 위에 PPO·GRPO·CISPO가 있다. GRPO 경로는 같은 프롬프트에서 $G$개 응답을 뽑아 그룹 안에서 보상을 표준화한다.

$$
A_i = \frac{r_i - \mu_{g}}{\sigma_{g} + \epsilon}
$$

$\mu_g$와 $\sigma_g$는 한 그룹의 평균과 표준편차(비편향 아님)이고 $\epsilon = 10^{-4}$다. 정책 손실은 PPO 클리핑 형태에 참조 모델 대비 KL 항을 더한다.

$$
L = -\frac{1}{|o|}\sum_{t} \left[ \min\left(\rho_t A, \mathrm{clip}(\rho_t, 1-\varepsilon, 1+\varepsilon) A\right) - \beta D_{KL,t} \right]
$$

CISPO 경로는 비율을 위쪽에서만 자르고(`clamp(ratio, max=epsilon_high)`) 그래디언트를 살린다. 두 경로 모두 KL 계수 $\beta$와 클리핑 폭 $\varepsilon$을 인자로 노출한다. 여기서 방법론적으로 중요한 것은 **보상이 하나의 스칼라가 아니라 항의 합**이라는 점이다. 일반 대화 데이터에서는 응답 길이 구간(20-800자, 벗어나면 -0.5), 사고 구간 길이(20-300자, +1.0), 사고 닫힘 태그 1회(+0.25), 반복 패널티, 그리고 보상 모델 점수를 더한다. 그 합을 자르는 것은 에이전트 경로이고, 그때 총합이 [-3, 3] 밖으로 나가지 않는다.

$$
r = \mathrm{clip}\left(r_{len} + r_{think} + r_{close} + r_{RM} - r_{rep}, -3, 3\right)
$$

에이전트 경로(`train_agent.py`)는 이 위에 다중 턴 궤적을 얹는다. 턴 수 상한 3, 턴당 생성 256 토큰, 도구 응답은 2,048자로 잘라 넣고, 턴별 태그 균형 어긋남의 절대합에 0.5를 곱해 깎으며, 정답 도구 집합과 호출 집합의 차이를 벌점으로 쓴다.

$$
\text{gap} = \left| n_{valid} - \lvert g \rvert \right| + \max\left(0, n_{call} - n_{valid}\right)
$$

즉 "필요한 도구를 다 불렀는가"와 "쓸데없는 도구를 더 불렀는가"를 분리해 벌점화한다. 이 분리가 없으면 모델은 도구를 많이 부르는 쪽으로 퇴화한다.

### E. 서빙과 평가

`serve_openai_api.py`는 OpenAI 호환 엔드포인트를 띄우고 `reasoning_content`, `tool_calls`, `open_thinking` 필드를 지원한다. `web_demo.py`는 Streamlit 채팅 UI로 사고 과정과 도구 선택을 보여준다. 평가는 `lm-evaluation-harness`를 그대로 쓰고, 지시 모델에는 `--apply_chat_template`을 붙인다. 이 저장소의 평가 절이 특이한 이유는 **점수보다 점수의 해상도**를 먼저 말한다는 점이다. 데이터셋별 표본 수와 표준오차(obqa ±1.9, ceval ±1.2, cmmlu ±0.4)를 함께 적고, 두 모델의 차이는 표준오차의 약 $\sqrt{2}$배이므로 obqa에서 5점 미만, ceval·piqa·siqa에서 3점 미만 차이는 모델을 구분하지 못한다고 못 박는다.

| Model | params | zh (ceval / cmmlu) | en (arc / piqa / obqa / hellaswag / siqa) |
| :--- | :---: | :--- | :--- |
| minimind-3 | 64M | 24.89 / 25.38 | 28.49 / 50.65 / 23.60 / 28.28 / 34.19 |
| minimind-3-moe | 198M | 25.48 / 24.32 | 27.74 / 50.71 / 26.20 / 27.43 / 34.03 |
| minimind-3-exam | 64M | 30.98 / 26.12 | 35.61 / 56.26 / 24.20 / 28.40 / 34.19 |
| SmolLM2-135M | 135M | 24.44 / 24.71 | 58.50 / 68.17 / 32.80 / 43.15 / 39.46 |
| TinyLlama-1.1B | 1.1B | 25.71 / 25.03 | 54.80 / 74.43 / 35.60 / 60.38 / 43.09 |

`minimind-3-exam`은 더 큰 모델이 아니다. 같은 64M에 선택지 형식 정렬용 LoRA를 한 번 얹어 합친 것이고, 그 결과 7개 과제 평균이 약 2.9포인트 올랐다. 저자가 강조하는 해석은 "작은 모델의 병목이 지식이 아니라 **입력 형식 정렬**일 수 있다"는 것이다. 오염 여부도 명시한다. 정렬 데이터와 평가 데이터는 교집합이 없고, 오염된 하위 집합에서는 97%가 나왔지만 그 수치는 의미가 없다고 적는다.

---

## 3. 구조적 비교 (Comparative Positioning)

| 항목 | nanoGPT류 최소 구현 | 소형 공개 모델(SmolLM2, TinyLlama) | HuggingFace TRL 스택 | MiniMind |
| :--- | :--- | :--- | :--- | :--- |
| 사전학습 코드 | ✓ | ✗ (가중치만) | 부분 | ✓ |
| 대화·지시 학습 | ✗ | ✓ (가중치) | ✓ | ✓ (코드+데이터) |
| 선호·강화학습 | ✗ | ✗ | ✓ (추상화) | ✓ (DPO/PPO/GRPO/CISPO 직접 구현) |
| 도구 호출·에이전트 RL | ✗ | ✗ | 부분 | ✓ (다중 턴 롤아웃 포함) |
| 구조 최신성 | GPT-2 수준 | 계열별 상이 | 무관 | Qwen3 계열 정렬 |
| 단일 GPU 전체 재현 | ✓ (사전학습만) | ✗ | ✗ | ✓ (3090, 단계별 비용 공개) |
| 평가 해상도 공개 | ✗ | ✗ | ✗ | ✓ (표준오차·오염 여부) |
| 산출물의 이전성 | 낮음 | 높음 | 높음 | 높음 (transformers/llama.cpp/vLLM) |

`llm/` 안에서의 위치는 "응용 앱 모음"도 "프론티어 모델"도 아닌 **교육·재현용 최소 전체 스택**이다. 같은 카테고리의 응용 저장소가 "모델로 무엇을 만드는가"를 보여준다면, 이 저장소는 "모델이 무엇으로 만들어지는가"를 코드 한 줄 단위로 보여준다.

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **점수는 우연 근처다**: ceval 24.89는 4지선다 우연 수준(25)과 구분되지 않는다. 이 모델의 가치는 성능이 아니라 절차에 있다. 성능으로 인용하면 오독이다.
2. **영어가 구조적으로 약하다**: 학습 비중이 중국어에 치우쳐 있고 어휘가 6,400개로 작아, 영어 과제에서 동급 SmolLM2-135M과 20포인트 이상 벌어진다.
3. **깊이 대 너비의 자기모순**: 문서는 MobileLLM의 "같은 파라미터에서 깊은 쪽이 유리하다"를 근거로 인용하면서, 정작 64M 주력은 8층을 택한다. 저자는 학습 속도와 안정성의 공학적 타협이라고 밝히지만, 그 타협의 비용(얕은 모델의 상한)은 측정되지 않았다.
4. **MoE의 학습 비용**: 네이티브 PyTorch 구현에서 4 experts/top-1은 dense보다 약 50% 느리다. "MoE는 추론이 빠르다"는 통념이 학습 단계에서는 뒤집힌다는 사실을 스스로 인정한다.
5. **정렬 세금(Alignment Tax)**: 도구 호출 과제에서 full_sft 60%에서 agent 85%로 오르지만, 같은 가중치가 사실 질문에서 더 자신 있게 틀린다. 저자가 직접 "도구는 잘하지만 개방형 질문에서 더 잘 지어낸다"고 적는다.
6. **어휘 절충**: 6,400 토큰 어휘는 임베딩·출력층 비중을 줄이지만 중국어 인코딩 효율이 qwen2·glm 계열보다 나쁘다. 작은 모델일수록 이 절충이 유리하다는 판단이지만, 다국어 용도에는 부적합하다.
7. **증류의 성격**: 주력 SFT 데이터는 강모델 출력에 대한 지도학습(블랙박스 증류)에 가깝다. 교사 모델의 정책과 데이터 라이선스가 산출물의 상한과 법적 제약을 함께 결정한다.
8. **분산 규모의 증거 부족**: DDP·DeepSpeed 지원을 표방하지만, 다수 노드에서의 처리량·안정성 곡선은 공개되어 있지 않다.
9. **평가 범위**: 7개 선다형 과제와 소규모 도구 호출 20문항이 전부다. 대화 품질·안전성·장문 일관성은 측정되지 않는다.
10. **버전 드리프트**: 데이터 파일 이름과 설정 기본값이 릴리스마다 바뀌고(minimind2 → minimind-3), 문서의 일부 표는 과거 버전을 함께 싣는다. 재현 시 커밋 해시를 고정해야 한다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **회계 가능한 최소 모델**: 설정에서 파라미터 수를 다시 계산해 확인하는 습관은 모델 카드 신뢰 문제에 대한 실무적 답이다. 같은 방식으로 깊이·너비·어휘 크기의 교환 곡선을 작은 예산에서 실측할 수 있다.
- **보상 분해의 교훈**: 길이·형식·반복·모델 점수를 항으로 나누고 총합을 자르는 설계는, 단일 보상 모델이 만드는 해킹 경로를 줄인다. 도구 호출에서 "누락"과 "과호출"을 분리해 벌점화한 것도 같은 원리다.
- **평가 해상도의 문서화**: 표준오차와 오염 여부를 표와 함께 공개하는 관행은, 작은 모델 논문에서 재현 가능성과 해석 가능성을 동시에 올린다. 순위를 주장하기 전에 구분 가능성을 먼저 계산하는 태도가 표준이 되어야 한다.
- **형식 정렬이라는 렌즈**: 같은 가중치에 선택지 형식 LoRA만 얹어 평균 2.9포인트를 얻은 실험은, 벤치마크 점수가 지식량이 아니라 인터페이스 적합도를 재는 경우가 많다는 것을 보여준다.
- **교육용 커리큘럼의 구조**: 토크나이저 → 사전학습 → SFT → LoRA → DPO → RL → 증류이 순서는 각 단계가 이전 단계의 산출물을 입력으로 받는 의존 그래프다. 강의·온보딩 설계에도 그대로 쓸 수 있는 순서다.
- **후속 연구 방향**: (i) 8층과 16층의 동일 예산 비교, (ii) 6,400 어휘의 언어별 비용 곡선, (iii) 도구 호출 보상에서 과호출 벌점의 민감도, (iv) 증류 데이터 오염이 평가에 미치는 영향의 정량화, (v) MoE 학습에서 fused kernel 도입 전후의 처리량 비교.
