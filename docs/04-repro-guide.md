# 04. 실험·재현 가이드

> 기준 문서: [vendor/Adaptive-RAG/README.md](../vendor/Adaptive-RAG/README.md)  
> **정직한 전제:** 전체 파이프라인은 **GPU**, **대용량 디스크**, **Elasticsearch**, **다운로드 데이터**가 필요합니다.  
> 분류기 학습만 돌리거나, upstream 제공 tarball로 일부 단계를 건너뛸 수 있습니다.

## 0. 재현 수준 선택

| 수준 | 필요 리소스 | 가능한 것 |
|------|-------------|-----------|
| **A. 문서·코드 읽기** | 없음 | 아키텍처 이해 |
| **B. tarball 활용** | CPU, ~10GB+ 디스크 | 전처리 데이터·prediction·classifier data로 후반부 재현 |
| **C. 전체 재현** | GPU( FLAN-T5-XL/XXL ), ES, ~수십 GB | dev/test 3전략 전부 실행 → classifier → eval |

아래는 **C 전체 재현** 기준이며, B는 해당 단계에서 tarball 사용으로 대체합니다.

## 1. 환경 (Environment)

```bash
cd vendor/Adaptive-RAG

conda create -n adaptiverag python=3.8
conda activate adaptiverag

pip install torch==1.13.1+cu117 \
  --extra-index-url https://download.pytorch.org/whl/cu117
pip install -r requirements.txt
```

| 항목 | upstream 요구 |
|------|---------------|
| Python | 3.8 |
| PyTorch | 1.13.1+cu117 (CUDA 11.7) |
| Elasticsearch | 7.10.2 (client lib 7.9.1) |
| GPU | FLAN-T5-XL/XXL 로컬 서빙 시 **필수에 가깝음** |
| OpenAI | `MODEL=gpt` 사용 시 `OPENAI_API_KEY` |

> Cloud Agent VM 등 GPU 없는 환경에서는 **llm_server 기동·대규모 predict가 불가**합니다.  
> 이 경우 `predictions.tar.gz`로 prediction 단계를 대체하세요.

## 2. Retriever 서버

### Elasticsearch

```bash
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-linux-x86_64.tar.gz
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.10.2-linux-x86_64.tar.gz.sha512
shasum -a 512 -c elasticsearch-7.10.2-linux-x86_64.tar.gz.sha512
tar -xzf elasticsearch-7.10.2-linux-x86_64.tar.gz
cd elasticsearch-7.10.2/
./bin/elasticsearch   # port 9200
```

### Retriever API

```bash
cd vendor/Adaptive-RAG
uvicorn serve:app --port 8000 --app-dir retriever_server
```

## 3. 데이터 (Datasets)

### Option B: upstream tarball (권장 shortcut)

```bash
cd vendor/Adaptive-RAG
tar -xzf processed_data.tar.gz   # 전처리 QA 데이터
tar -xzf predictions.tar.gz      # 3전략 prediction (선택)
tar -xzf data.tar.gz             # 분류기 데이터 (선택)
```

### Option C: 처음부터 다운로드

**Multi-hop** (MuSiQue, HotpotQA, 2WikiMultiHopQA):

```bash
bash ./download/processed_data.sh
bash ./download/raw_data.sh
python processing_scripts/subsample_dataset_and_remap_paras.py musique dev_diff_size 500
python processing_scripts/subsample_dataset_and_remap_paras.py hotpotqa dev_diff_size 500
python processing_scripts/subsample_dataset_and_remap_paras.py 2wikimultihopqa dev_diff_size 500
python retriever_server/build_index.py hotpotqa   # 데이터셋별 반복
```

**Single-hop** (NQ, TriviaQA, SQuAD): DPR raw + Wikipedia `psgs_w100.tsv` — upstream README의 wget 블록 참조.

```bash
python processing_scripts/process_nq.py
python processing_scripts/process_trivia.py
python processing_scripts/process_squad.py
python processing_scripts/subsample_dataset_and_remap_paras.py nq test 500
python retriever_server/build_index.py wiki
```

**Overlap 검사:**

```bash
python processing_scripts/check_duplicate.py nq
```

### 인덱스 확인

```bash
curl localhost:9200/_cat/indices
```

upstream README 기대치: HotpotQA ~5,233,329 · 2Wiki ~430,225 · MuSiQue ~139,416 · Wiki ~21,015,324 docs.

## 4. LLM 서버

```bash
MODEL_NAME=flan-t5-xl uvicorn serve:app --port 8010 --app-dir llm_server
# XXL: MODEL_NAME=flan-t5-xxl
# GPT: export OPENAI_API_KEY=... 후 MODEL=gpt 로 run script 사용
```

## 5. 세 가지 Retrieval 전략 실행

### Dev (classifier silver label용)

```bash
SYSTEM=ircot_qa    # multi: ircot_qa | single: oner_qa | zero: nor_qa
MODEL=flan-t5-xl
DATASET=nq
LLM_PORT_NUM=8010

bash run_retrieval_dev.sh $SYSTEM $MODEL $DATASET $LLM_PORT_NUM
```

6 datasets × 3 systems × model 조합 — **매우 많은 GPU 시간**.

### Test (최종 QA prediction)

```bash
bash run_retrieval_test.sh $SYSTEM $MODEL $DATASET $LLM_PORT_NUM
```

`predictions.tar.gz` 사용 시 이 단계 생략 가능.

## 6. Classifier 학습·추론

```bash
python ./classifier/preprocess/preprocess_predict.py
python ./classifier/preprocess/preprocess_step_efficiency.py

python ./classifier/preprocess/preprocess_silver_train.py flan_t5_xl
python ./classifier/preprocess/preprocess_silver_valid.py flan_t5_xl
python ./classifier/preprocess/preprocess_binary_train.py
python ./classifier/preprocess/concat_binary_silver_train.py

cd classifier
bash ./run/run_large_train_xl.sh
cd ..

python ./classifier/postprocess/predict_complexity_on_classification_results.py flan_t5_xl
```

`data.tar.gz` 사용 시 preprocess 일부 생략 가능 (upstream README).

> **주의:** `predict_complexity_on_classification_results.py` 내부에 **하드코딩된 classification result 경로**가 있습니다.  
> 학습 후 실제 output path에 맞게 수정해야 합니다.

## 7. 최종 평가

```bash
python ./evaluate_final_acc.py
```

`evaluate_final_acc.py`의 `base_pred_path`도 학습 run 경로에 맞게 수정 필요.

## 8. 리소스·리스크 체크리스트

| 항목 | 필요 여부 |
|------|-----------|
| NVIDIA GPU (16GB+ XL, 더 큼 XXL) | 로컬 LLM 시 **필수** |
| RAM 32GB+ | Elasticsearch + indexing |
| 디스크 50GB+ | Wikipedia index, checkpoints |
| 네트워크 | 데이터·모델 다운로드 |
| 시간 | full grid 수일~ (하드웨어 의존) |

## 9. GPU/데이터 없이 할 수 있는 것

1. `docs/` + `vendor/Adaptive-RAG` 코드 정독
2. `processed_data.tar.gz`만 풀어 데이터 스키마 확인
3. `predictions.tar.gz` + `data.tar.gz`로 classifier postprocess·eval 스크립트 경로 맞춘 **부분 재현**
4. 소규모 subsample로 **단일 dataset · 단일 system** smoke test (여전히 GPU 권장)

## 10. 문제 해결

| symptom | 확인 |
|----------|------|
| Retriever connection error | ES 9200, retriever 8000 기동 |
| LLM timeout | `llm_server` port, `.llm_server_address.jsonnet` |
| OOM | flan-t5-xxl → xl 축소, batch size config |
| Missing prediction | `run_retrieval_test.sh` 완료 또는 tarball |
| Wrong eval path | `evaluate_final_acc.py`, postprocess 내 path 수정 |

## 참고

- 공식 README: [vendor/Adaptive-RAG/README.md](../vendor/Adaptive-RAG/README.md)
- 코드 맵: [03-upstream-repo-map.md](03-upstream-repo-map.md)
- 논문: [arXiv:2403.14403](https://arxiv.org/abs/2403.14403)
