# 03. Upstream 저장소 맵 (starsuzi/Adaptive-RAG)

> 경로 기준: `vendor/Adaptive-RAG/` (git subtree)

## 루트 스크립트 — “무엇을 실행하면 무엇이 되나”

| 파일 | 용도 |
|------|------|
| `run_retrieval_dev.sh` | dev set(500 subsample)에서 3전략 실험 — **분류기 학습용 silver label 생성** |
| `run_retrieval_test.sh` | test set에서 3전략 실험 — **최종 QA prediction** |
| `runner.py` | `run.py` 래퍼: config write / predict / evaluate / summarize |
| `run.py` | 실험 본체 (대형) |
| `predict.py` | prediction 유틸 |
| `evaluate.py` | 단일 실험 evaluation |
| `evaluate_final_acc.py` | **Adaptive-RAG 최종 EM/F1** (분류기 merge 후) |
| `lib.py` | JSON/JSONL I/O, config path 헬퍼 |

### `run_retrieval_*.sh` 인자

```bash
bash run_retrieval_dev.sh SYSTEM MODEL DATASET LLM_PORT_NUM
# SYSTEM: ircot_qa | oner_qa | nor_qa  (+ ircot, oner variants)
# MODEL:  flan-t5-xl | flan-t5-xxl | gpt
# DATASET: nq | trivia | squad | hotpotqa | 2wikimultihopqa | musique
```

내부 흐름: `runner.py write` → `predict` → `evaluate` → `summarize`

## 디렉터리 맵

```
vendor/Adaptive-RAG/
├── base_configs/          # jsonnet 실험 설정 템플릿
├── classifier/            # 복잡도 분류기 (학습·후처리)
│   ├── preprocess/        # silver/binary 데이터 전처리
│   ├── run/               # t5-large 학습 shell
│   ├── postprocess/       # 분류 결과 → adaptive prediction merge
│   ├── run_classifier.py
│   └── utils.py
├── commaqa/               # IRCoT skeleton QA 프레임워크 (inference, execution)
├── download/              # 데이터 다운로드 shell
├── llm_server/            # FLAN-T5 FastAPI 서버
├── retriever_server/      # Elasticsearch BM25 retriever FastAPI
├── processing_scripts/    # raw → standard format, subsample, dedup check
├── prompts/               # 데이터셋별 프롬프트
├── prompt_generator/      # 프롬프트 생성 보조
├── metrics/               # EM/F1 등 metric 구현
├── official_evaluation/   # HotpotQA, MuSiQue 등 공식 eval 스크립트
├── images/                # 논문 figure (adaptiverag.png)
├── processed_data.tar.gz  # 전처리된 dev/test (upstream 제공)
├── predictions.tar.gz     # 3전략 prediction 결과 (upstream 제공)
├── data.tar.gz            # 분류기 학습 데이터 (upstream 제공)
├── requirements.txt
└── README.md              # 공식 설치·실험 가이드
```

## 주요 서브시스템

### 1. Retriever (`retriever_server/`)

| 항목 | 내용 |
|------|------|
| 실행 | `uvicorn serve:app --port 8000 --app-dir retriever_server` |
| 전제 | Elasticsearch 7.10.2 (port 9200) |
| 인덱스 | `build_index.py {dataset}` — hotpotqa, 2wikimultihopqa, musique, wiki |
| 기대 문서 수 (upstream README) | HotpotQA ~5.2M, 2Wiki ~430K, MuSiQue ~139K, Wiki ~21M |

### 2. LLM Server (`llm_server/`)

| 항목 | 내용 |
|------|------|
| 실행 | `MODEL_NAME=flan-t5-xl uvicorn serve:app --port 8010 --app-dir llm_server` |
| 모델 | flan-t5-xl, flan-t5-xxl (로컬 GPU), gpt (API key) |
| 설정 | `.llm_server_address.jsonnet`, `.retriever_address.jsonnet` |

### 3. Classifier (`classifier/`)

| 단계 | 스크립트 | 설명 |
|------|----------|------|
| Predict 전처리 | `preprocess/preprocess_predict.py` | QA performance 측정용 |
| Step efficiency | `preprocess/preprocess_step_efficiency.py` | step 수 효율 지표 |
| Silver train | `preprocess/preprocess_silver_train.py {flan_t5_xl\|xxl\|gpt}` | 정답 맞힌 전략 기반 |
| Silver valid | `preprocess/preprocess_silver_valid.py {model}` | silver validation |
| Binary train | `preprocess/preprocess_binary_train.py` | 데이터셋 inductive bias |
| Concat | `preprocess/concat_binary_silver_train.py` | 학습 데이터 병합 |
| Train | `run/run_large_train_xl.sh` (등) | T5-large fine-tune |
| Postprocess | `postprocess/predict_complexity_on_classification_results.py` | adaptive prediction 생성 |
| Eval | `../evaluate_final_acc.py` | 최종 accuracy |

### 4. 데이터 (`download/`, `processing_scripts/`)

| 스크립트 | 용도 |
|----------|------|
| `download/processed_data.sh` | multi-hop test 전처리 데이터 |
| `download/raw_data.sh` | multi-hop raw → dev 준비 |
| `processing_scripts/subsample_dataset_and_remap_paras.py` | 500 subsample |
| `processing_scripts/process_{nq,trivia,squad}.py` | single-hop 포맷 통일 |
| `processing_scripts/check_duplicate.py` | dev/test overlap 검사 |

## 전형적 실험 순서 (맵 관점)

```
1. conda env + pip install
2. Elasticsearch + retriever_server + llm_server 기동
3. 데이터 다운로드/전처리 또는 tar.gz 압축 해제
4. build_index.py
5. run_retrieval_dev.sh  × (3 systems × 6 datasets × models)  # 오래 걸림
6. run_retrieval_test.sh × (동일)
7. classifier/preprocess/* → classifier/run/*
8. classifier/postprocess/predict_complexity_on_classification_results.py
9. evaluate_final_acc.py
```

**단축 경로:** upstream이 제공하는 `processed_data.tar.gz`, `predictions.tar.gz`, `data.tar.gz`로 5–6단계 일부 생략 가능 (README 참조).

## 설정·프롬프트

- **`base_configs/`** — jsonnet 실험 config (retrieval count, distractor count 등)
- **`prompts/{dataset}/`** — NQ, HotpotQA 등 데이터셋별 prompt template
- **`commaqa/`** — QA inference/execution 코어 (IRCoT lineage)

## 이 스터디 저장소에서의 위치

| 목적 | 위치 |
|------|------|
| upstream 수정 없이 읽기 | `vendor/Adaptive-RAG/` |
| upstream 업데이트 | 루트 README의 `git subtree pull` |
| 한국어 설명 | `docs/` |

upstream 변경은 subtree pull로 반영하고, **스터디 노트만 `docs/`에 추가**하는 것을 권장합니다.
