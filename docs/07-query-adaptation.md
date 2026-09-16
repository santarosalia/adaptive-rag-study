# 07. 쿼리 적응 — user / missing / ADR 변천

> **출처:** [chat-agent ADR 0008](https://github.com/santarosalia/chat-agent/blob/main/docs/adr/0008-retrieve-query-rewrite.md) · [ADR 0011](https://github.com/santarosalia/chat-agent/blob/main/docs/adr/0011-retrieve-sufficiency-evaluator.md) · [Adaptive-RAG](https://arxiv.org/abs/2403.14403)

chat-agent에서 **“어떤 문자열로 hybrid RAG를 검색할 것인가”** 는 Adaptive-RAG 논문의 **complexity routing**과는 다른 축입니다. 여기서는 **쿼리 적응**만 정리합니다.

## 1. 현재 규칙 (ADR 0011, authoritative)

| retrieve 라운드 | `POST /v1/retrieve` 의 `query` |
|-----------------|--------------------------------|
| **1회 (시드)** | 대화에서 **뒤에서 가장 가까운 user 메시지 원문** |
| **2회 이후** | 직전 evaluate JSON의 **`missing: string[]`** 를 **공백 한 칸으로 join** |
| **`missing`이 빈 배열** | 다시 **그 user 원문**으로 검색 |

관련 state ([ADR 0011](https://github.com/santarosalia/chat-agent/blob/main/docs/adr/0011-retrieve-sufficiency-evaluator.md)):

- **`searchHistory`**: 라운드마다 query·결과 기록. 길이 = 라운드 수
- **`evaluationHistory`**: 평가기 출력 이력 → 다음 evaluate 프롬프트 입력

평가기 출력 예:

```json
{
  "sufficient": false,
  "missing": ["B의 적용 조건", "B의 예외 사항"],
  "confidence": 0.72
}
```

→ 2번째 retrieve `query` ≈ `"B의 적용 조건 B의 예외 사항"`

## 2. Adaptive-RAG와의 대비

| | 논문 `ircot_qa` | chat-agent |
|---|-----------------|------------|
| 추가 검색 쿼리 | IRCoT가 **CoT 중간 단계**에서 생성 | 평가기 **`missing`** (부족 **정보 항목** 명시) |
| 트리거 | complexity **C** (사전 분류) | **`sufficient: false`** (사후 판단) |
| 상한 | IRCoT 설정 | **3회 retrieve**, `top_k` 5→10→20 |

논문은 **질문 전체**에 대한 retrieval **전략**을 고르고, chat-agent는 **같은 user 턴 안**에서 **부족한 facet**을 채우는 **쿼리 적응**에 가깝습니다.

## 3. ADR 0008 — 대화 기반 리라이트 (역사)

[ADR 0008](https://github.com/santarosalia/chat-agent/blob/main/docs/adr/0008-retrieve-query-rewrite.md) (**Superseded**):

- **문제:** “그거”, “며칠이야”처럼 **지시어**만 있는 user utterance는 hybrid 검색에 불리
- **안:** retrieve **전** 별도 LLM으로 **독립 검색 질문 한 줄** 생성
  - 입력: 최근 최대 **6턴** (user+assistant), assistant는 맥락용 **앞 400자**
  - 첫 턴: 리라이트 LLM **호출 안 함**, user 원문 그대로
  - 실패: user 원문 폴백
- **Contract A 유지:** RAG `/v1/query` 위임 없음 ([ADR 0001](https://github.com/santarosalia/chat-agent/blob/main/docs/adr/0001-contract-a-retrieve-only.md))

0008은 [ADR 0010](https://github.com/santarosalia/chat-agent/blob/main/docs/adr/0010-retrieve-as-answer-tool.md) → [ADR 0011](https://github.com/santarosalia/chat-agent/blob/main/docs/adr/0011-retrieve-sufficiency-evaluator.md)으로 대체되며, **현재 그래프에는 “retrieve 전 LLM 리라이트” 노드가 없습니다.**

## 4. ADR 0011 — missing 기반 적응 (현재)

[ADR 0011](https://github.com/santarosalia/chat-agent/blob/main/docs/adr/0011-retrieve-sufficiency-evaluator.md)이 0008/0010을 대체한 이유 요약:

| 이슈 | 0010 (답변 LLM 도구) | 0011 (평가기) |
|------|----------------------|---------------|
| 추가 검색 트리거 | 모델 tool call (건너뛰기 쉬움) | **서버** evaluate 분기 |
| 다음 쿼리 | 모델 임의 | **`missing` join** (규칙 고정) |
| 대화 지시어 | 리라이트 LLM (0008)에 의존 | 1회는 **user 원문**; 지시어는 평가기가 `missing`으로 **구체화** 기대 |

**트레이드오프:** 멀티턴 지시어를 **사전에** 풀지 않고 user 원문으로 첫 검색 → 평가기·추가 라운드에 의존. 0008처럼 “검색 질문 한 줄”을 미리 만드는 비용은 없음.

## 5. end-to-end 예 (스터디용)

**User:** “아까 말한 B 정책, 예외는?”

1. **retrieve #1** — query = user 원문 전체. `top_k=5`
2. **evaluate** — `sufficient: false`, `missing: ["B 정책 예외 조항", "B 정책 적용 범위"]`
3. **retrieve #2** — query = `"B 정책 예외 조항 B 정책 적용 범위"`. `top_k=10`
4. **evaluate** — `sufficient: true` → **answer** (누적 citations + answer LLM)

3회 retrieve 후에도 `sufficient: false`면 **그 citations로 answer** ([ADR 0011](https://github.com/santarosalia/chat-agent/blob/main/docs/adr/0011-retrieve-sufficiency-evaluator.md)).

## 6. truncate와의 관계 ([ADR 0004](https://github.com/santarosalia/chat-agent/blob/main/docs/adr/0004-context-truncate.md))

- **truncate (N=20)** 는 **답변 LLM 입력** (`prepare`)에만 적용
- **retrieve `query`** 는 truncate 대상 **아님** — 1회 user 원문, 2회+ `missing` join
- RAG가 가져온 `[Retrieved context]`는 truncate **후** answer system에 붙음

## 다음 읽을 문서

- [06-langgraph-pipeline.md](06-langgraph-pipeline.md)
- [05-paper-vs-chat-agent.md](05-paper-vs-chat-agent.md)
