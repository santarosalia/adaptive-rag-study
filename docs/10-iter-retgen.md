# 10. Iter-RetGen — Iterative Retrieval-Generation Synergy

> **논문 제목:** Enhancing Retrieval-Augmented Large Language Models with Iterative Retrieval-Generation Synergy  
> **출처:** [arXiv:2305.15294](https://arxiv.org/abs/2305.15294) · Findings of EMNLP 2023  
> **방법명:** **Iter-RetGen** (논문 표기: ITER-RETGEN)

> **식별 참고:** arXiv `2305.15294`는 **FLARE** ([2305.06983](https://arxiv.org/abs/2305.06983))가 **아닙니다**.  
> FLARE는 “Active Retrieval Augmented Generation”으로 **생성 중** low-confidence token 기준 active retrieval입니다.  
> 본 논문은 **전체 출력을 iteration 단위**로 retrieval ↔ generation을 반복하는 **Iter-RetGen**입니다.

## 1. 문제 정의 (Problem)

Retriever는 특히 **복잡한 정보 필요**(multi-hop QA 등)에서 query–document **semantic gap**을 잘 못 메웁니다.  
최근 연구는 LLM **generation**을 retrieval에 끼워 **relevance modeling**을 개선합니다.

Iter-RetGen은 “**모델의 (중간) 출력이 과업 완료에 무엇이 필요한지 드러낸다**”는 관찰에서 출발해, 그 출력을 **다음 retrieval의 context**로 쓰는 **반복 retrieval–generation synergy**를 제안합니다.

## 2. 알고리즘 (Iter-RetGen)

질문 `q`, corpus `D`, 최대 iteration `T`:

**iteration `t` (1 ≤ t ≤ T):**

1. **Retrieve:** query = `y_{t-1} || q` (첫 iteration은 `y_0` 없음 → **`q`만**). top-`k` paragraph 검색 → `D_{y_{t-1}||q}`.
2. **Generate:** LLM `M`에 `q`와 검색된 paragraph **전체**를 prompt에 넣어 출력 `y_t` 생성.

최종 응답 = **`y_T`**.

```
y_t = M( y_t | prompt( D_{y_{t-1}||q}, q ) ),  1 ≤ t ≤ T
```

### 2.1 Generation-augmented retrieval

- 1차 retrieval: **질문만**으로 검색 → answer recall이 낮을 수 있음 (논문 Table 6).
- 2차 이후: **이전 LLM 출력**이 prerequisite sub-question·facet을 드러내 **검색 쿼리를 풍부화**.
- IRCoT·ReAct·Self-Ask처럼 **출력 생성과 retrieval을 token/token interleave**하지 않고, **iteration마다 retrieved knowledge 전체**를 prompt에 넣어 **생성 유연성**을 유지합니다 (논문이 interleaved 방식과 대비).

### 2.2 Generation-augmented retrieval adaptation (선택)

- re-ranker `φ`는 `(q, d)` relevance + **1차 출력 `y_1` 접근** 가능.
- retriever query encoder만 distillation해, **추론 시에는 user input만**으로도 개선된 retrieval 기대.
- HotpotQA·Feverous 등에서 adaptation ablation 보고.

## 3. 실험 설정 (개요)

- **Task:** multi-hop QA (HotpotQA, 2WikiMultiHopQA, MuSiQue 2-hop, Bamboogle), fact verification (Feverous), commonsense (StrategyQA).
- **Baseline:** Direct/CoT prompting, ReAct, Self-Ask, DSP 등.
- **Prompting:** few-shot **chain-of-thought** (Wei et al. 스타일); retrieval-augmented CoT 반복.
- **효율:** 논문은 `T=2`, iteration당 top-5 retrieval 등 설정에서 ReAct·Self-Ask 대비 **적은 API call·적은 retrieved paragraph**로 competitive/superior 성능을 주장합니다.

> **주의:** Table 2·3의 Acc/EM/F1, API call 수 등 **구체적 수치는 논문 표를 직접 인용**하세요.  
> 이 스터디 문서는 수치를 임의로 기재하지 않습니다.

## 4. Adaptive-RAG와의 비교

| 관점 | Adaptive-RAG ([2403.14403](https://arxiv.org/abs/2403.14403)) | Iter-RetGen ([2305.15294](https://arxiv.org/abs/2305.15294)) |
|------|---------------------------------------------------------------|----------------------------------------------------------------|
| **적응 축** | 질문 **complexity** → no/single/multi | **iteration 횟수** + 이전 **생성물**로 query 확장 |
| **multi-step** | **IRCoT** (CoT 중간 쿼리) | **전체 출력** `y_{t-1}` + `q`로 retrieve |
| **결정 시점** | retrieve **전** classifier | **매 iteration** 동일 규칙 (고정 `T` 또는 early stop 없음 — 논문 기본) |
| **no retrieval path** | **A → nor_qa** | 없음 (항상 retrieval-augmented prompt; direct prompting은 baseline) |
| **학습** | complexity **classifier** fine-tune | **prompting** (+ 선택적 retriever distillation) |
| **평가 환경** | 6 QA benchmark grid | multi-hop + verification + commonsense |

**공통점:** 복잡 질문에서 **1회 retrieval 부족** 문제를 **다단계**로 해결.  
**차이:** Adaptive-RAG는 **사전 routing** + IRCoT; Iter-RetGen은 **사후 생성물**로 retrieval query를 **점진적 보강**.

## 5. chat-agent와의 비교

| 관점 | Iter-RetGen | chat-agent ([ADR 0011](https://github.com/santarosalia/chat-agent/blob/main/docs/adr/0011-retrieve-sufficiency-evaluator.md)) |
|------|-------------|-----------------------------------------------------------------------------------------------------------------------------------|
| **다음 retrieval context** | **이전 LLM 전체 출력** `y_{t-1}` | **평가기 `missing[]`** join ([07-query-adaptation.md](07-query-adaptation.md)) |
| **루프 트리거** | 고정 **iteration `T`** | **`sufficient: false`** (최대 3 retrieve) |
| **중간 생성 역할** | **검색 쿼리 힌트** (정답 보장 없음) | **부족 facet 명시** (evaluator LLM) |
| **답변** | 마지막 iteration **단일 LLM** 출력 | **answer LLM** (retrieve 도구 없음) |
| **대화** | 단일 QA 질문 `q` | **멀티턴** user utterance + truncate |

**개념적 유사:** 둘 다 “1차 context로 부족하면 **더 targeted query**로 재검색” — Adaptive-RAG **C / ircot_qa** 및 chat-agent **evaluate loop**와 같은 **multi-step IR** 계열.

**차이:** Iter-RetGen은 **evaluator 없이** 생성물 자체를 query augmentation으로 사용; chat-agent는 **structured sufficiency critique** + **명시적 상한**.

## 6. FLARE ([2305.06983](https://arxiv.org/abs/2305.06983))와 한 줄 구분

| | Iter-RetGen (본 문서) | FLARE |
|---|----------------------|-------|
| **단위** | **iteration** (전체 `y_t`) | **sentence / token** (upcoming sentence 예측) |
| **retrieve trigger** | 매 iteration (또는 고정 `T`) | **low-confidence token** in draft sentence |
| **task 초점** | QA·verification·commonsense | **long-form** knowledge-intensive generation |

## 7. 스터디 질문 (자가 점검)

1. Iter-RetGen 2차 retrieval query는? → **`y_1 || q`** (1차 LLM 출력 + 원 질문).
2. Adaptive-RAG `ircot_qa`와의 차이? → IRCoT는 **CoT step interleave**; Iter-RetGen은 **batch prompt에 knowledge 전체**.
3. chat-agent `missing`과 `y_{t-1}`의 차이? → `missing`은 **evaluator가 구조화한 gap**; `y_{t-1}`은 **free-form reasoning+answer draft**.

## 다음 읽을 문서

- [07-query-adaptation.md](07-query-adaptation.md) — chat-agent 쿼리 적응
- [11-adaptive-rag-family-comparison.md](11-adaptive-rag-family-comparison.md) — 계열 통합 비교
- [01-paper-summary.md](01-paper-summary.md) — Adaptive-RAG 본 논문
