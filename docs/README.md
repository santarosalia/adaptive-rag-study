# Adaptive-RAG 스터디 문서

한국어 중심 학습 노트입니다. 수치·실험 결과는 [논문](https://arxiv.org/abs/2403.14403) 및 [upstream README](../vendor/Adaptive-RAG/README.md)를 출처로 합니다.

## 목차

1. [논문 요약](01-paper-summary.md)
2. [아키텍처](02-architecture.md)
3. [Upstream 코드 맵](03-upstream-repo-map.md)
4. [재현 가이드](04-repro-guide.md)
5. [논문 vs chat-agent](05-paper-vs-chat-agent.md)
6. [chat-agent LangGraph 파이프라인](06-langgraph-pipeline.md)
7. [쿼리 적응 (user / missing)](07-query-adaptation.md)
8. [Self-RAG](08-self-rag.md)
9. [CRAG](09-crag.md)
10. [Iter-RetGen (2305.15294)](10-iter-retgen.md)
11. [Adaptive RAG 계열 통합 비교](11-adaptive-rag-family-comparison.md)
12. [용어집](glossary.md)

## 읽는 순서 (권장)

```
01-paper-summary → 02-architecture → 03-upstream-repo-map → 04-repro-guide
                                      ↘ 05-paper-vs-chat-agent → 06-langgraph-pipeline → 07-query-adaptation
                                      ↘ 08-self-rag → 09-crag → 10-iter-retgen → 11-adaptive-rag-family-comparison
                                      ↘ glossary (필요 시)
```

**계열 논문만 빠르게:** `08` → `09` → `10` → `11` (Adaptive-RAG·chat-agent 맥락은 `01`·`05` 선행 권장)

## 출처

- 논문: [Adaptive-RAG (arXiv:2403.14403)](https://arxiv.org/abs/2403.14403)
- 계열 논문: [Self-RAG (2310.11511)](https://arxiv.org/abs/2310.11511) · [CRAG (2401.15884)](https://arxiv.org/abs/2401.15884) · [Iter-RetGen (2305.15294)](https://arxiv.org/abs/2305.15294)
- 코드: [starsuzi/Adaptive-RAG](https://github.com/starsuzi/Adaptive-RAG) (Apache-2.0, `vendor/Adaptive-RAG/`)
- 프로덕션 지향 구현: [santarosalia/chat-agent](https://github.com/santarosalia/chat-agent) (05–07, 11에서 ADR 기준으로 연결)
