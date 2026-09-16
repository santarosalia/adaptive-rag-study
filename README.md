# adaptive-rag-study

[Adaptive-RAG](https://arxiv.org/abs/2403.14403) 논문과 [starsuzi/Adaptive-RAG](https://github.com/starsuzi/Adaptive-RAG) 공식 구현을 함께 읽기 위한 **한국어 중심 스터디 저장소**입니다.

> **Adaptive-RAG: Learning to Adapt Retrieval-Augmented Large Language Models through Question Complexity**  
> Soyeong Jeong, Jinheon Baek, Sukmin Cho, Sung Ju Hwang, Jong C. Park — NAACL 2024  
> arXiv: [2403.14403](https://arxiv.org/abs/2403.14403)

## 이 저장소의 역할

| 구분 | 설명 |
|------|------|
| `docs/` | 논문 요약, 아키텍처, upstream 코드 맵, 재현 가이드 (한국어) |
| `vendor/Adaptive-RAG/` | upstream 공식 코드 ([starsuzi/Adaptive-RAG](https://github.com/starsuzi/Adaptive-RAG), **Apache-2.0**) |
| 루트 `NOTICE` | 서드파티 코드 출처·라이선스 고지 |

**이 저장소는 논문/코드의 학습·정리용이며, upstream의 공식 배포판이 아닙니다.** 실험 재현·인용·배포 시에는 반드시 원 논문과 upstream 저장소를 참조하세요.

## Upstream 반영 방식 (git subtree)

upstream 코드는 **git subtree**로 `vendor/Adaptive-RAG/`에 포함했습니다. Apache-2.0 조건에 따라 upstream `LICENSE` 파일과 저작권 고지를 그대로 유지합니다.

```bash
# subtree 최초 추가 (이미 완료됨)
git subtree add --prefix=vendor/Adaptive-RAG \
  https://github.com/starsuzi/Adaptive-RAG.git main

# 이후 upstream 동기화
git fetch adaptive-rag-upstream main
git subtree pull --prefix=vendor/Adaptive-RAG adaptive-rag-upstream main
```

- **Upstream remote:** `adaptive-rag-upstream` → `https://github.com/starsuzi/Adaptive-RAG.git`
- **Import commit:** `0c88670af8707667eb5c1163151bb5ce61b14acb`
- **라이선스:** [Apache License 2.0](vendor/Adaptive-RAG/LICENSE) — 상세 고지는 [NOTICE](NOTICE)

## 스터디 문서

| 문서 | 내용 |
|------|------|
| [docs/01-paper-summary.md](docs/01-paper-summary.md) | 논문 문제 정의, 복잡도 분류기, 3가지 전략, 결과 |
| [docs/02-architecture.md](docs/02-architecture.md) | Adaptive-RAG vs naive RAG 아키텍처 |
| [docs/03-upstream-repo-map.md](docs/03-upstream-repo-map.md) | upstream 디렉터리·스크립트 맵 |
| [docs/04-repro-guide.md](docs/04-repro-guide.md) | 환경·데이터·학습·평가 재현 가이드 |
| [docs/glossary.md](docs/glossary.md) | 용어집 |

## 빠른 시작 (upstream 코드)

upstream README와 동일한 흐름입니다. 상세·주의사항은 [docs/04-repro-guide.md](docs/04-repro-guide.md)를 참고하세요.

```bash
cd vendor/Adaptive-RAG
conda create -n adaptiverag python=3.8
conda activate adaptiverag
pip install torch==1.13.1+cu117 --extra-index-url https://download.pytorch.org/whl/cu117
pip install -r requirements.txt
```

## 인용 (Citation)

논문:

```bibtex
@inproceedings{jeong2024adaptiverag,
  author    = {Soyeong Jeong and Jinheon Baek and Sukmin Cho and
               Sung Ju Hwang and Jong Park},
  title     = {Adaptive-RAG: Learning to Adapt Retrieval-Augmented Large Language
               Models through Question Complexity},
  booktitle = {NAACL},
  year      = {2024},
  url       = {https://arxiv.org/abs/2403.14403}
}
```

upstream 코드:

```bibtex
@misc{starsuzi_adaptiverag,
  author       = {Soyeong Jeong and Jinheon Baek and Sukmin Cho and
                  Sung Ju Hwang and Jong Park},
  title        = {Adaptive-RAG Official Code},
  howpublished = {\url{https://github.com/starsuzi/Adaptive-RAG}},
  year         = {2024},
  note         = {Apache-2.0}
}
```

## 라이선스

- **`vendor/Adaptive-RAG/`** — [Apache License 2.0](vendor/Adaptive-RAG/LICENSE) (upstream 저작권 유지)
- **`docs/` 및 루트 README·NOTICE** — 스터디 목적의 원본 문서 (upstream 코드와 별도)

서드파티 고지 전문: [NOTICE](NOTICE)
