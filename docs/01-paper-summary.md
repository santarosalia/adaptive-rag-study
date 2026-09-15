# 01. 논문 요약 — Adaptive-RAG

> **출처:** [arXiv:2403.14403](https://arxiv.org/abs/2403.14403) · NAACL 2024  
> **공식 코드:** [starsuzi/Adaptive-RAG](https://github.com/starsuzi/Adaptive-RAG)

## 1. 문제 정의 (Problem)

Retrieval-Augmented Generation(RAG)은 외부 지식을 LLM에 주입해 QA 정확도를 높이는 대표적 방법입니다. 그러나 **질문 복잡도는 데이터셋·사용자마다 다릅니다.**

| 접근 | 한계 |
|------|------|
| 항상 retrieval (single/multi-step) | 단순 질문에도 검색·추론 비용이 과도함 |
| retrieval 없음 (parametric only) | 다단계(multi-hop) 질문에서 부족 |
| 고정된 adaptive retrieval | 단순/복잡 이분법에 치우치거나, 복잡 질문 처리가 불충분 |

논문은 **질문마다 최적의 RAG 전략을 동적으로 선택**하는 프레임워크를 제안합니다. 목표는 **정확도와 효율(검색·추론 step 수)의 균형**입니다.

## 2. 핵심 아이디어: Query Complexity Classifier

Adaptive-RAG는 incoming query의 **복잡도 수준**을 예측하는 **작은 LM 분류기**(구현: T5-large)를 사용합니다.

### 자동 라벨 수집 (Automatic Labeling)

수동 라벨링 대신 두 가지 신호로 학습 데이터를 구성합니다.

1. **Silver labels** — 동일 LLM으로 세 가지 retrieval 전략을 dev/test에 실행한 뒤, **정답을 맞힌 전략**을 해당 질문의 복잡도 proxy로 사용
2. **Binary labels (inductive bias)** — 데이터셋 고유 특성(예: single-hop vs multi-hop 데이터셋 출처)에서 얻는 이진 신호

두 소스를 concat하여 분류기 학습 데이터를 만듭니다 (`classifier/preprocess/`).

### 복잡도 → 전략 매핑 (3단계)

논문·코드에서 전략은 대략 다음과 대응합니다.

| 복잡도 라벨 | 전략 | upstream `SYSTEM` | 설명 |
|-------------|------|-------------------|------|
| A (low) | **No retrieval** | `nor_qa` | LLM parametric knowledge만 사용 |
| B (medium) | **Single-step retrieval** | `oner_qa` | 1회 검색 후 답변 |
| C (high) | **Multi-step retrieval** | `ircot_qa` | IRCoT 기반 반복 검색·추론 |

분류기가 예측한 라벨에 따라 **해당 전략의 prediction만** 최종 답변으로 선택합니다.

## 3. 실험 설정 (개요)

- **QA 벤치마크 (6개):**
  - Single-hop: Natural Questions (NQ), TriviaQA, SQuAD
  - Multi-hop: HotpotQA, 2WikiMultiHopQA, MuSiQue
- **백본 LLM:** FLAN-T5-XL/XXL, GPT (OpenAI API) 등
- **Retriever:** Elasticsearch + BM25 (`retriever_server/`)
- **비교:** 고정 single/multi/zero 전략, adaptive retrieval baseline 등

구체적 하이퍼파라미터·인덱스 크기는 upstream README 및 `base_configs/`를 참고하세요.

## 4. 결과 (Results)

논문 abstract 및 본문의 주장(출처: [arXiv:2403.14403](https://arxiv.org/abs/2403.14403)):

- 다양한 복잡도를 아우르는 open-domain QA에서 **전체 정확도와 효율을 동시에 개선**
- 관련 adaptive retrieval baseline 대비 **균형 잡힌(balanced) 전략** 제공
- 단순 질문에는 불필요한 multi-step overhead를 줄이고, 복잡 질문에는 충분한 retrieval depth 확보

> **주의:** EM/F1, step efficiency 등 **구체적 수치는 논문 표·그래프를 직접 인용**하세요.  
> 이 스터디 문서는 수치를 임의로 기재하지 않습니다.  
> upstream은 `predictions.tar.gz`, `evaluate_final_acc.py`로 재현·검증 경로를 제공합니다.

## 5. 기여 요약

1. **Query complexity**에 따른 3단계 RAG 전략 자동 선택
2. **Silver + binary** 자동 라벨로 분류기 학습 (수동 annotation 최소화)
3. 6개 QA 데이터셋에서 **accuracy–efficiency trade-off** 개선 입증

## 6. 관련 링크

- 논문 PDF: https://arxiv.org/pdf/2403.14403.pdf
- LlamaIndex / LangGraph / ReAct 예제 (upstream README 참조)
- IRCoT skeleton: https://github.com/StonyBrookNLP/ircot

## 다음 읽을 문서

- [02-architecture.md](02-architecture.md) — 파이프라인 구조
- [03-upstream-repo-map.md](03-upstream-repo-map.md) — 코드 맵
- [04-repro-guide.md](04-repro-guide.md) — 재현 절차
