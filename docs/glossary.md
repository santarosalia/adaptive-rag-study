# 용어집 (Glossary)

Adaptive-RAG 스터디에서 자주 쓰는 용어입니다. 코드 identifier는 영문 그대로 표기합니다.

| 용어 | 설명 |
|------|------|
| **RAG** | Retrieval-Augmented Generation. 검색된 문서를 LLM context에 넣어 답변 생성 |
| **Adaptive-RAG** | 질문 복잡도에 따라 no / single / multi retrieval을 선택하는 QA 프레임워크 |
| **Query complexity** | 질문이 요구하는 추론·검색 깊이. 논문에서 A/B/C (또는 low/medium/high)로 분류 |
| **nor_qa** | No retrieval QA — `SYSTEM` 이름, retrieval 없이 LLM만 사용 |
| **oner_qa** | One retrieval QA — single-step RAG |
| **ircot_qa** | IRCoT 기반 multi-step retrieval QA |
| **IRCoT** | Interleaving Retrieval with Chain-of-Thought. 반복 검색·추론 QA 방법 |
| **Silver label** | 세 전략 실행 결과, **정답을 맞힌 전략**을 해당 query의 complexity proxy로 사용한 라벨 |
| **Binary label** | 데이터셋 출처 등 **inductive bias**에서 얻는 이진 complexity 신호 |
| **Classifier** | T5-large 기반 query complexity 예측 모델 (`classifier/`) |
| **BM25** | Elasticsearch 기본 lexical retriever (upstream) |
| **Single-hop QA** | 1-hop 질문 (NQ, TriviaQA, SQuAD) |
| **Multi-hop QA** | 여러 문서·단계가 필요한 질문 (HotpotQA, 2WikiMultiHopQA, MuSiQue) |
| **EM / F1** | Exact Match / token F1 — QA 정확도 metric |
| **Step efficiency** | 답변당 retrieval·reasoning step 수 — 효율 지표 |
| **Dev subsample (500)** | classifier 학습용 dev subset (`dev_diff_size 500`) |
| **jsonnet** | 실험 config 작성에 쓰는 설정 언어 (`base_configs/`) |
| **git subtree** | upstream 저장소를 하위 경로에 병합하는 git 방식 (`vendor/Adaptive-RAG/`) |

## 복잡도 라벨 ↔ 전략 (코드)

| 라벨 | 전략 | `SYSTEM` |
|------|------|----------|
| A | No retrieval | `nor_qa` |
| B | Single-step | `oner_qa` |
| C | Multi-step | `ircot_qa` |

## 약어

| 약어 | 풀네임 |
|------|--------|
| NQ | Natural Questions |
| ES | Elasticsearch |
| LLM | Large Language Model |
| QA | Question Answering |
