# 📈 inference-optimisation: The Ceiling Before the Flag — Byte-Counted Rooflines, Interleaved Control–Candidate–Control A/B, and a Long-Context Gate for Local LLM Serving

- **Repository**: [sybil-solutions/inference-optimisation-skill](https://github.com/sybil-solutions/inference-optimisation-skill)
- **Stars**: ⭐ 2,156 (forks 205, created 2026-11-16, last commit 2026-11-16)
- **Category**: `Skills` / Serving-Speed Hill-Climbing Skill & Measurement Protocol
- **Project**: Sybil Solutions; distilled from published hill-climbs in `local-ai-registry`, `trellis-serve` and `qwen-serve`; loads as a Claude Code skill or as a plain prompt
- **License / Runtime**: MIT / Python standard library plus bash; benchmarks any OpenAI-compatible server (vLLM, SGLang, llama.cpp server, TabbyAPI, MLX-LM, LM Studio, Ollama `/v1`); hardware-agnostic in principle, validated on Intel Xe, NVIDIA and Apple silicon
- **Key Themes**: theoretical ceilings from measured bandwidth and FLOPs, bytes per emitted token with the routed-expert union, interleaved control–candidate–control A/B with medians, an append-only experiment ledger, long-context needle and concurrent full-KV gates, pinned clean builds with six acceptance gates

---

## 1. 연구 배경 및 시스템 병목 (Systems Research Problem)

로컬 LLM 서빙의 속도 조정은 실험 설계 문제이지 플래그 암기 문제가 아니다. 엔진 문서가 권하는 조합은 특정 카드·특정 모델·특정 버전에서 얻은 결과이고, 다른 조합에 그대로 옮기면 이득이 사라지거나 손실이 된다. 이 스킬이 먼저 겨냥하는 것은 최적화가 아니라 **측정의 네 가지 실패**다.

1. **기준선의 부재**: 53 tok/s가 좋은 값인지 나쁜 값인지 판정하려면 그 기계·그 모델의 상한이 필요하다. 상한이 없으면 개선의 여지가 남았는지, 사용자가 요구한 목표가 물리적으로 가능한지 말할 수 없다. 이 스킬은 최적화 전에 상한을 계산하고, 목표가 상한을 넘으면 산술과 함께 먼저 알린다.
2. **측정 오류**: 투기적 디코딩을 켜면 엔진이 SSE 청크 하나에 여러 토큰을 담아 보내므로 청크를 세면 처리량이 과소 집계된다. "32K"라고 부른 프롬프트가 실제로는 58K였던 사례, 통합 메모리에서 14 GiB로 보인 점유가 실제 29.4 GiB였던 사례가 문서에 남아 있다. 계측 자체가 실험의 일부다.
3. **교란**: 실행 간 잡음은 ±5–10%다. 한 시간 간격의 같은 설정이 53.8과 49.5로 갈렸고, 배터리 구동과 밸런스 프로파일은 8–10%를, 가벼운 CPU 부하는 노트북 GPU 디코드를 107에서 75 tok/s로 떨어뜨렸다. 단발 비교로 얻은 "승리"의 상당 부분은 표류다.
4. **속도만 보고 통과시키는 관행**: 빠른 결과가 깨진 출력일 수 있다. 117 tok/s 기록은 번호 목록을 반복 출력한 모델이었다. 긴 컨텍스트에서 검색이 실패해도 짧은 구간의 속도는 멀쩡하다. 그래서 자격 판정이 속도 측정과 분리된 단계로 존재한다.

이 네 실패는 각각 다른 처방을 요구한다. 상한은 대역폭·FLOPs 회계로, 측정 오류는 계측 규약으로, 교란은 교차 배치와 중앙값으로, 안정성은 게이트로 다룬다. 다음 절은 그 처방들을 하나의 절차로 묶는 방법을 다룬다.

---

## 2. 아키텍처 및 수학적 정식화 (Architecture & Formulation)

### A. 세 부분과 그 역할

| 부분 | 파일 | 줄 수 | 역할 |
| :--- | :--- | ---: | :--- |
| 절차 | `SKILL.md` | 166 | 인벤토리, 상한, 원칙 10개, 루프, 레버 순서, 정지·자격·보고 |
| 참조 | `references/ceilings.md`, `methodology.md`, `levers.md`, `gotchas.md`, `publishing.md` | 79·55·87·58·64 | 상한 계산, 측정 함정, 레버별 실측 근거, 엔진·하드웨어 함정, 자격·배포 |
| 하네스 | `bench.py`, `ceiling.py`, `queue.sh`, `mem.sh` | 262·51·14·31 | OpenAI 호환 서버 벤치, 상한 계산기, 직렬 큐, 통합 메모리 점유 측정 |
| 프롬프트 | `PROMPT.md`, `README.md` | 94·151 | 스킬을 로드하지 못하는 에이전트용 동일 절차, 결과 요약 |

참조 문서는 필요할 때만 읽는다. 상한식을 쓰지 않는 단계에서 `ceilings.md`를 컨텍스트에 올릴 이유가 없고, 레버의 실측 근거는 레버를 고르는 순간에만 필요하다. 표의 파일을 합치면 1,112줄이고 그중 절차 본문이 166줄이라는 비율이 이 스킬의 성격을 보여준다. 판단은 모델이 하고, 스킬은 **판단의 순서와 증거 요건**을 고정한다.

### B. 디코드 상한: 토큰당 바이트

디코드는 메모리 대역폭에 묶인다. 토큰 하나를 내보내는 데 읽는 바이트를 체크포인트 텐서 크기에서 직접 세고, 측정된 대역폭으로 나눈다.

$$
B_{\text{tok}} = B_{\text{dense}} + B_{\text{head}} + B_{\text{exp}} \frac{k}{E} + b_{\text{kv}} c + 2 b_{\text{state}}
$$

$k/E$는 routed expert 중 활성 비율, $b_{\text{kv}}$는 토큰당 KV 바이트, $c$는 컨텍스트, 마지막 항은 SSM·GDN 순환 상태의 읽기·쓰기다. 주의할 점이 둘 있다. 출력 헤드가 어휘가 큰 모델에서 전체의 10–25%를 차지하고, 하이브리드 모델의 KV는 층 수가 아니라 **KV를 가진 층 수**로 계산해야 한다.

투기적 디코딩을 켜면 검증 한 번이 읽는 expert는 각 행의 합집합이다. $d$개 드래프트, 동시성 $N$, 수락 길이 $a$에 대해

$$
B_{\text{verify}} = B_{\text{dense}} + B_{\text{head}} + B_{\text{exp}} \left(1 - \left(1 - \frac{k}{E}\right)^{\left(d+1\right) N}\right)
$$

$$
\text{tok/s} = \frac{a W}{B_{\text{verify}} + B_{\text{draft}}},
\quad B_{\text{draft}} \approx d N \left(b_{\text{layer}}^{\text{draft}} + b_{\text{head}}^{\text{draft}}\right)
$$

여기서 두 가지가 따라온다. 첫째, 드래프트의 출력 헤드는 종종 전체 드래프트 비용의 최대 항이므로 **어휘를 잘라야** 한다. 둘째, 수락률이 올라도 스텝 시간이 더 크게 오르면 손해다. 스킬이 수락 길이와 스텝 시간을 분리해 기록하라고 요구하는 이유다.

### C. 프리필 상한: 계산량

프리필은 계산에 묶인다. 토큰당 FLOPs는 활성 선형 파라미터와 attention 항의 합이고, 실용 상한은 측정 피크의 약 50%다.

$$
F_{\text{tok}} \approx 2 P_{\text{act}} + 4 L_{\text{attn}} H d \bar{p}
$$

$\bar{p}$는 평균 위치다. causal attention에서 평균 위치는 $L/2$이므로 항은 $4 L_{\text{attn}} H d \cdot p$ 형태로 위치에 비례해 커진다. 여기서 상한을 낮추는 주범은 FLOPs가 아니라 **GEMM 효율**이다. MoE에서 청크당 expert별 행 수가 적으면 grouped GEMM의 타일이 낭비되고, 그 손실이 계산식에 나타나지 않는다. 스킬이 "패딩 절벽"을 별도 레버로 두는 이유다.

### D. TTFT, 기동, KV 용량

| 지표 | 바닥 | 구성 |
| :--- | :--- | :--- |
| 짧은 프롬프트 TTFT | 순방향 1회 + 토크나이즈 + 스케줄 | 노트북 30–60 ms, 데이터센터 GPU 수 ms |
| 기동 | 가중치 바이트 ÷ 디스크·페이지 캐시 속도 + 할당 | JIT·autotune·graph 캐시가 남아 있으면 약 10초 |
| KV 용량 | 스트림 수 $N_{\max}$ | 아래 식 |

$$
N_{\max} = \frac{M_{\text{cap}} - M_{\text{w}} - M_{\text{s}} - M_{\text{r}}}{c \cdot n_{\text{kv}} \cdot 2 \cdot h_{\text{kv}} \cdot d_{\text{head}} \cdot b_{\text{elem}}}
$$

분모가 토큰당 KV 바이트다. 이 값을 계산하고 나면 "이 기계가 4명 × 256K를 실제로 감당하는가"가 측정 전에 답해진다. 실제 확인은 동시 전장 요청 소크로 한다.

### E. 통계 계약: 교차 배치와 중앙값

스킬의 실험 규칙은 통계적 주장이 아니라 **판정 절차**다. 디코드 샘플 6개 이상의 중앙값을 쓰고, 지원자 설정은 양쪽 통제 사이에서 실행하며, 다음을 모두 만족할 때만 채택한다.

$$
M_{\text{cand}} > \max\left(M_{\text{ctl}}^{\text{pre}}, M_{\text{ctl}}^{\text{post}}\right),
\quad \min_{m} \frac{x_{m}^{\text{cand}}}{x_{m}^{\text{ctl}}} \ge 0.97
$$

즉 목표 지표에서 양쪽 이웃을 모두 이기고, 어떤 지표도 3% 넘게 후퇴시키지 않아야 한다. 엔진끼리 비교할 때는 같은 카드·같은 날 연속 실행하고, 중앙값 비율의 부트스트랩 95% 신뢰구간이 1.0을 넘을 때만 "B가 빠르다"고 말한다. 표류가 측정 대상보다 클 수 있으므로 비교는 **이웃끼리만** 성립한다.

### F. 레버 순서와 탐욕 선택

루프는 상한 격차와 적용 가능성의 곱이 큰 레버를 먼저 시도한다. 순서는 비용이 낮고 효과가 큰 것부터다.

| 순위 | 레버 | 범위 |
| :--- | :--- | :--- |
| 1 | JIT·컴파일·graph 캐시 영속화 | 기동 50–70% 감소 |
| 2 | 디코드 그래프(CUDA·XPU graph capture) | 단독으로 c1 2배 수준 |
| 3 | 투기적 디코딩(내장 MTP·EAGLE·드래프트 모델·n-gram) | 동시성·컨텍스트별 드래프트 길이 스윕 |
| 4 | 동시성·KV 풀 크기, 하이브리드 상태 슬롯, 프리픽스 캐시 | 스트림 수와 TTFT |
| 5 | 프리필 청크 크기, attention 라이브러리, int8·fp8 GEMM | 처리량 대 디코드 정지의 맞바꿈 |
| 6 | 커널 스레드·타일 크기 | 기본값이 큰 카드 기준인 경우가 많음 |
| 7 | 커널 자체 수정 | 프로파일과 ISA를 읽은 뒤에만 |

정지 판정도 규칙이다. 연속 세 개의 레버가 잡음 안에 들면 플레이트로 선언하고, `PLAN.md`에 최고값·기준값·상한, 남은 병목의 근거, 남은 레버의 비용 추정을 적는다. "그만둘 시점"을 측정으로 정의하는 것이 이 스킬의 실질적 기여 중 하나다.

### G. 게이트: 무엇을 "최고"라고 부를 수 있는가

속도 개선은 자격을 통과해야 최고가 된다. 긴 컨텍스트 검색(약 50%, 이어서 약 100% 지점의 바늘을 정확히 회수), 동시 전장 요청 소크(KV 풀이 N명을 전제하면 필수), 무손실을 주장하는 변경의 greedy 동등성 검사. 배포 전에는 깨끗한 체크아웃에서 핀으로 고정해 빌드하고 여섯 개 수용 게이트를 다시 통과시킨다.

| 게이트 | 확인 내용 |
| :--- | :--- |
| Load | 서버가 healthy, `/v1/models`에 모델이 보임 |
| Chat | 짧은 답변이 stop으로 종료 |
| Reasoning | 사고와 본문이 분리되고 정답이 맞음 |
| Tools | 도구 호출 왕복 후 반환값의 코드를 인용 |
| Context | 창의 약 85% 지점 바늘을 정확히 회수 |
| Speed | 첫 토큰 후 30초 구간 디코드가 하한 이상 |

게이트 수치와 힐클라임 수치는 비교하지 않는다. 사고 모드·측정 구간·하네스가 다르기 때문이며, 보고서에 어느 수치가 어디서 나왔는지 적는 것이 규칙이다.

### H. 기록 구조

실험 디렉터리는 `PLAN.md`(타임스탬프가 붙은 모든 실험과 판정·이유), `THEORY.md`(상한), `ledger.jsonl`(실행당 한 줄, 하네스가 기록), `configs/`(실행당 JSON 하나), `best-config.json`으로 고정된다. 하네스는 토큰 수를 서버 사용량 통계에서만 가져오고, 반복 출력·요청 실패·배터리 구동을 플래그로 남기며, 메모리 상한 초과 시 실행을 종료한다. 실패를 적는 것이 규칙인 이유는 그것이 가장 재사용 가능한 산출물이기 때문이다.

---

## 3. 정량 벤치마크 (Quantitative Evidence)

### A. 힐클라임 결과

| 하드웨어 | 모델·양자화·엔진 | 이전 → 이후 |
| :--- | :--- | :--- |
| Intel Arc B390 iGPU (64 GB LPDDR5X, 약 115 GB/s) | Ornith-1.5-35B-A3B, EXL3 4bpw, SGLang XPU + MTP | 디코드 1명 43.6 → 53.0 tok/s, 4명 84.7 → 103, 프리필 32K 1,037 → 1,660, 256K 297 → 628, 기동 113 → 43초 |
| Intel Arc Pro B70 | Qwen3.8-27B, EXL3 4bpw, vLLM + XPU 커널 | 디코드 91.2(1명) / 365(16명), 같은 카드 llama.cpp Q4_K_M 25.0, 프리필 4K·32K·128K·254K 1,654·1,434·928·650 → 2,421·2,259·1,415·911 |
| Intel Arc Pro B70 | 같은 모델, SGLang | 디코드 82.6 대 vLLM 75.7(같은 날) |
| RTX 3090 | Qwen3.8-27B EXL3 3bpw + MTP | 96.2(산문) / 141.1(코드) |
| RTX 3090 | Qwen3.6-35B-A3B EXL3 3bpw + MTP | 251.7 / 357.9 |
| 4× H200 NVL | DeepSeek-V4.1-Flash, vLLM + 투기 | 부하에서 크래시 → 229.1 디코드, 17,546 프리필 |

표에서 읽어야 할 것은 배수 자체가 아니라 **같은 카드에서 엔진과 커널이 만드는 차이**다. B70에서 llama.cpp 25.0과 vLLM 91.2는 3.6배이고, 같은 vLLM 위에서도 SGLang이 9% 앞선다. 반면 4× H200 행은 최적화가 실패에서 출발했음을 보여준다.

### B. B390의 상한 대 실적

이 스킬이 자란 실행이다. 측정 하드웨어는 115 GB/s 복사 대역폭(스펙 153.6 GB/s)과 4096³에서 35.8 TFLOPS(8192³에서 20)다. 모델은 40층 중 30층이 GDN 선형 attention이고 256 expert 중 8개를 고른다.

| 구성 요소 | GB/토큰 |
| :--- | ---: |
| Routed expert 16.26 GB × 8/256 | 0.508 |
| Dense(GDN 0.515 + full attention 0.137 + shared 0.064 + router 0.042) | 0.758 |
| 출력 헤드(6비트, 248K 어휘) | 0.382 |
| 합계 | 1.648 |

| 모드 | GB/산출 토큰 | 상한 |
| :--- | ---: | ---: |
| 투기 없음 | 1.68 | 68 tok/s |
| MTP 1개, 전체 드래프트 헤드, 수락 1.85 | 1.38 | 83 tok/s |
| MTP 1개, 32K로 가지친 드래프트 헤드 | 1.19 | 97 tok/s |
| 드래프트 2개, 수락 약 2.5 | 1.11 | 약 103 tok/s |

실적은 53 tok/s로 상한의 약 55%다. 남은 격차의 두 원인이 문서에 명시된다. 모델의 MTP 헤드가 깊이 2에서 약 14%만 수락해 두 번째 드래프트 행에 도달할 수 없었고, 나머지는 양자화 GEMV 커널이 검증 그래프에서 약 76 GB/s까지만 냈기 때문이다. 프리필은 256K에서 실용 상한 670–684 대비 628 tok/s로 상한에 근접했다. KV는 FP8에서 토큰당 10,240바이트, 256K 스트림당 2.7 GB이며 46 GiB 상한에서 4개 전장 스트림이 들어간다.

### C. 레버별 실측 근거

| 레버 | 측정 |
| :--- | :--- |
| 디코드 그래프 | B70 14.4 → 28.4, SGLang XPU 16 → 32, vLLM INT4 11 → 27 |
| 드래프트 헤드 어휘 가지치기 | B70 37.9 → 48.2(+27%), 1,940블록 중 512개로 98.6% 커버 |
| 프리픽스 캐시 | 66K 2턴 TTFT 36 → 0.45초, 32K 2턴 29 → 0.95초 |
| 프리필 청크 축소 | B70 8,192 → 2,048에서 최악 정지 7.1 → 1.95초, 디코드 −4% |
| int8 프리필 GEMM | 선형 1.89배, NLL +0.17%, top-1 일치 97.2%, 디코드 비트 동일 |
| MoE GEMM 패딩 회피 | 64행 M-블록으로 프리필 +7%, 비트 동일, NT=2 타일로 c16 +22% |
| 기동 캐시 | B390 113 → 43초, 4×RTX PRO 6000 vLLM 22분 → 7분 |

이 표의 값들은 서로 다른 기계에서 나온 것이므로 이식 가능한 약속이 아니라 **사전 확률**로 쓰라는 단서가 붙는다. 스택별 태그가 붙은 이유다.

### D. 투기적 디코딩이 지는 경우

수락률이 높아도 느릴 수 있다는 사실이 정량으로 제시된다.

| 사례 | 결과 |
| :--- | :--- |
| 7토큰 드래프트(수락 3.26) 대 MTP(2.85) | 11–14% 느림 |
| Apple M4, MTP 3개 | 6.35 → 3.25 tok/s |
| llama.cpp MTP, 어려운 프롬프트 | 36.7 대 45.4 |
| 검증을 프리필 커널로 보낼 때 | 1K 컨텍스트마다 스텝 +3.5 ms, 16K에서 47 → 104 ms |
| CPU 샘플러 | 스텝당 약 3 ms, GPU로 옮기면 c1 +10% |

이 다섯 행이 이 스킬의 태도를 요약한다. 범용 처방은 없고, "스텝 시간과 수락 길이를 분리해 재고, 지원자만 통제와 비교하라"는 절차만 남는다.

---

## 4. 한계점 및 실패 모드 (Limitations & Trade-offs)

1. **상한은 상한일 뿐 목표가 아니다**: 실용 상한은 측정 피크의 절반이라는 가정 위에 서고, 그 계수가 커널·행렬 크기·클럭에 따라 움직인다. 상한 대비 55%라는 수치는 "남은 45%를 얻을 수 있다"는 뜻이 아니며, 문서도 나머지를 커널 효율로 지목할 뿐 회수 방법을 약속하지 않는다.
2. **하드웨어별 검증 범위의 비대칭**: Intel Xe에서 얻은 함정(분할 K의 캐시 펜스, 넓은 입력의 top-k 세그폴트, 피스와이즈 graph 캡처 실패)이 문서의 큰 부분을 차지한다. NVIDIA와 Apple은 상대적으로 얇고, AMD는 ROCm graph 누수 한 건과 ExLlama 수준이다.
3. **수락률 통계의 편향**: 일부 엔진은 요청이 끝날 때만 수락 카운터를 갱신한다. 짧은 관측 창은 표본이 치우치고, 문서는 전체 요청으로 재라고 경고한다. 계측기 자체의 신뢰도가 항상 확보되지는 않는다.
4. **드래프트의 내용 의존성**: 수락률은 텍스트 종류와 온도에 따라 달라진다. 고정 패널을 쓰라는 규칙은 비교 가능성을 지키지만, 패널이 대표하지 않는 워크로드(코드 편집, 표, 다국어)에서는 패널의 결론이 뒤집힐 수 있다.
5. **단일 GPU 독점을 전제**: 큐가 GPU 하나를 잠그고 한 번에 한 작업만 돌린다. 여러 사용자가 공유하는 서버나 다중 임차 환경에서는 이 전제가 깨지고, 잡음 통제의 근거도 함께 약해진다.
6. **게이트의 비용**: 약 100% 컨텍스트 바늘 검사와 동시 전장 소크는 실행 시간이 길고, 긴 프리필 도중 강제 종료는 GPU 드라이버 폴트를 낼 수 있다. 검증이 싸지 않으므로 실제로는 생략될 유인이 생긴다.
7. **패치 의존 재현성**: SGLang XPU 백엔드의 여러 결함은 업스트림이 아니라 플러그인에서 고쳐졌다. 문서가 그 사실을 밝히지만, 게시된 레시피가 비공개 패치에 의존하면 제3자의 재현은 그만큼 어려워진다. 핀 고정과 깨끗한 빌드 규칙이 이를 완화할 뿐 없애지는 못한다.
8. **커널 작업의 진입 비용**: ISA를 읽고 레지스터 배치를 고치는 단계는 자동화되지 않는다. "스레드 4,096 → 1,024로 코드 c1 48.5 → 56.9" 같은 결과는 사람이 어셈블리를 읽은 뒤에 나온다.
9. **측정 함정의 목록은 닫혀 있지 않다**: 문서의 함정은 발견된 것이고, 새 엔진 버전마다 새 함정이 생긴다. 특히 투기 디코딩과 하이브리드 상태 저장은 릴리스마다 동작이 바뀐다.
10. **스킬은 자동 튜너가 아니다**: 하네스는 측정과 기록만 하고, 레버 선택과 해석은 에이전트의 판단에 맡긴다. 판단이 나쁘면 절차를 지켜도 나쁜 결론에 도달하며, 그 경우에도 게이트가 잡는 것은 속도가 아니라 붕괴뿐이다.
11. **결과의 출처 편향**: 보고된 큰 이득은 대부분 저자들이 직접 최적화한 스택에서 나왔다. 성공 사례만 표에 오르고, 실패한 힐클라임의 전체 분포는 공개되지 않는다. 사전 확률로 읽으라는 단서가 이 편향을 인정하는 장치다.
12. **비밀·환경 관리의 부담**: 자격 증명을 한 번만 묻고 출력·커밋·경로에 남기지 않는 규칙, 전원·프로파일 복원, 서버 종료 절차까지 스킬이 떠안는다. 자동화가 늘어날수록 실험 환경의 부작용 관리가 절차의 일부가 된다.

---

## 5. 연구 및 실무 활용 포인트 (Research Impact)

- **상한 우선 평가**: "우리 구성에서 53 tok/s"가 아니라 "이 기계의 상한 97 중 53"이라고 말하는 규율은 서빙 논문과 벤치마크 보고 전반에 필요한 형식이다. 상한식은 하드웨어와 모델 크기에서 유도되므로 다른 카드로 옮겨도 비교 가능한 좌표를 준다.
- **바이트당 토큰 회계의 이식성**: routed expert 합집합, 드래프트 헤드, KV, SSM 상태를 한 식에 모으면 새 모델·새 가속기에서 무엇이 병목일지 측정 전에 추정할 수 있다. 이 식은 디코드가 대역폭에, 프리필이 계산에 묶인다는 사실을 전제로 하며, 그 전제가 깨지는 구간(예: 아주 작은 배치의 통합 메모리)에서 한계가 드러난다.
- **교차 배치가 최소 기준선**: 표류가 시간당 측정 효과보다 큰 환경에서 단발 A/B는 정보를 주지 않는다. 통제–지원자–통제 배치와 중앙값 6개, 3% 후퇴 금지는 실험실 밖에서도 그대로 쓸 수 있는 최소 요건이다.
- **부정 결과의 자산화**: 거절된 실험과 그 이유를 남기는 관행은 같은 탐색을 반복하지 않게 하고, 나중에 하드웨어가 바뀌었을 때 재시도의 근거를 만든다. 최적화에서 가장 재사용되는 산출물이 실패 목록이라는 관점은 다른 도메인의 실험 관리에도 적용된다.
- **스텝 시간과 수락 길이의 분리**: 투기적 디코딩을 "수락률 최대화"로 다루는 통념을 정량으로 반박한다. $a$와 스텝 시간을 따로 재면 어느 드래프트가 이득인지 즉시 판정된다.
- **게이트를 자격으로 두는 설계**: 속도 개선과 안정성 판정을 분리하고, 게시 전 깨끗한 빌드에서 같은 게이트를 다시 통과시키는 절차는 "최적화된 첨부 설정"이 재현 불가능한 유물이 되는 것을 막는다.
- **에이전트 스킬의 형태**: 절차 166줄, 필요할 때 읽는 참조, 표준 라이브러리만 쓰는 하네스라는 구성은 지식 팩과 실행 도구의 경계를 보여준다. 모델이 판단하고 스킬이 증거 요건을 강제하는 분업은 측정이 중요한 다른 도메인(성능, 비용, 컴플라이언스)으로 옮길 수 있다.
- **후속 연구 방향**: (i) 커널 효율을 반영한 상한 보정 계수의 실험적 추정, (ii) 워크로드별 수락률 분포를 예측하는 드래프트 길이 정책, (iii) 다중 사용자 환경에서 교차 배치를 유지하는 실험 설계, (iv) 게이트 비용을 낮추는 표본 추출 검증, (v) 플러그인 패치 없이 업스트림에서 재현되는 레시피의 비율 측정.

### 참고 링크

- 저장소: https://github.com/sybil-solutions/inference-optimisation-skill
- 절차 본문: https://github.com/sybil-solutions/inference-optimisation-skill/blob/main/skills/inference-optimisation/SKILL.md
- 상한 계산: https://github.com/sybil-solutions/inference-optimisation-skill/blob/main/skills/inference-optimisation/references/ceilings.md
- 측정 방법론: https://github.com/sybil-solutions/inference-optimisation-skill/blob/main/skills/inference-optimisation/references/methodology.md
- 레버 근거: https://github.com/sybil-solutions/inference-optimisation-skill/blob/main/skills/inference-optimisation/references/levers.md
- 자격·배포: https://github.com/sybil-solutions/inference-optimisation-skill/blob/main/skills/inference-optimisation/references/publishing.md
- 원 실험 기록: https://github.com/sybil-solutions/local-ai-registry , https://github.com/sybil-solutions/trellis-serve
