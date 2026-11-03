# LLM 정규화 파트 — 구현하는 AI 에게 주는 전달사항

> 이 문서는 LLM 쪽 정규화 코드를 작성하는 AI(또는 사람)를 위한 지시문이다. 작업 전에 끝까지 읽고 지킨다.
> 작성 2026-11-03, 이헌준(정규화 Jev 담당). LLM 담당 지승환.

## 0. 목표

같은 정규화 문제를 **Jev(이미 구현됨)** 와 **LLM(네가 구현할 것)** 이 각각 풀고, **같은 채점기로 나란히 비교**한다.
정규화 = 「프로필에 적힌 표기 하나가 표준 목록의 어느 실체와 같은가, 또는 표준 목록에 없는가(NONE)」.

비교가 성립하려면 **모델에 들어가는 것과 채점 방식은 Jev 쪽과 한 글자도 다르면 안 되고, 모델을 부르는 방식만 다르다.**
이 원칙을 어기는 편의(프롬프트에 힌트 추가, 후보 재정렬, 지시문 다듬기 등)는 하지 않는다. 고치고 싶으면 Jev 쪽과 같이 고친다.

## 1. 먼저 읽을 것 (같은 폴더)

- `README.md` — 데이터 규격 · 결과 규격
- `common.py` — `load_data`, `build_candidates` (**수정 금지**)
- `jev_normalize.py` — `describe`, `build_request`, `run_one` (**수정 금지**, import 해서 쓴다)
- `score.py` — 채점기 (**수정 금지**)
- `RESULTS_1103.md` — Jev 1차 결과

## 2. 반드시 Jev 와 같아야 하는 것

| # | 항목 | 지키는 방법 |
|---|---|---|
| 1 | 데이터와 분할 | `common.load_data(<data_dir>)` 로 읽는다. `split` 칸을 바꾸지 않는다 |
| 2 | 후보와 순서 | `common.build_candidates(m, standard, k=…)` 결과를 **그대로**. 정렬 · 필터 · 중복 제거 금지. `k` 는 Jev 실행과 같은 값(기본 `None` = 그 유형 표준 목록 전체, 비교용 `5`) |
| 3 | 맥락(state) | `jev_normalize.build_request(m, cands, model="")["state"]` 를 그대로 프롬프트에 넣는다. 기간 겹침은 이미 코드가 계산해 후보 설명에 들어 있다 — 날짜 계산을 모델에 시키지 않는다 |
| 4 | 지시문 | `build_request(...)["questions"]["same_fwd"]["instructions"]` 를 **그대로**. 의역 · 보충 금지 |
| 5 | 선택지 | `["questions"]["same_fwd"]["criteria"]` 의 키(entity_id) 와 값(설명)을 **그 순서 그대로**. 마지막 키는 `NONE` 이다. 선택지 수 n 이 Jev 와 같아야 한다 |
| 6 | 확신도 | `confidence = (p_max − 1/n) / (1 − 1/n)` — TypeSafe 공식 식. n = 선택지 수(NONE 포함). 답 `pred_id` = 확률 최대 선택지 |
| 7 | 결과 칸 | 아래 4절 규격 그대로 (jsonl, 한 줄 = mention 하나) |
| 8 | 채점 | `python3 score.py <jev결과> <llm결과> --price-in … --price-out …` — 채점 로직을 따로 만들지 않는다 |
| 9 | 순서 · 반복 실험 | 하려면 `["questions"]["same_rev"]["criteria"]`(같은 후보 역순)로 한 번 더 물어 `pred_id_rev` 등에 넣는다. 반복은 `repeat` 칸 0, 1, … |

## 3. 네가 정하는 것 (LLM 쪽 자유)

- 프롬프트 형식 — 위 state · 지시문 · 선택지를 담기만 하면 JSON 그대로든 문장으로 풀든 상관없다. 시스템 프롬프트에는 「선택지 라벨 하나만 답하라」 정도만
- 모델 · `temperature=0`
- **확률은 logprobs 로 얻는다.** 선택지마다 한 토큰짜리 라벨(A, B, C …)을 붙이고, 답의 첫 토큰 위치의 `top_logprobs` 에서 각 라벨의 확률을 읽어 entity_id 로 되돌린다. 목록에 없는 라벨은 확률 0, 그다음 **합이 1 이 되게 정규화**한다
- **LLM 이 말로 쓴 확신도(「0.9」 등)는 쓰지 않는다** — 과신 경향이라 비교가 무의미해진다
- `any_match`(「후보 중 같은 게 있나」) 는 선택. 하려면 예/아니오 토큰 확률로 `noul_any`(예 확률) 칸을 채운다

## 4. 결과 규격 (`out/llm_<config>_<MMDD-HHMM>.jsonl`)

한 줄 예:

```json
{"mention_id": "M0001", "entity_type": "project", "surface": "차세대 여신 시스템 구축", "gold_id": "P04",
 "variant": "고객사 생략", "missing_client": "0", "split": "dev",
 "system": "llm", "config": "k=all", "model": "<실제 모델 ID>",
 "cand_ids": ["P04", "P01", "..."], "gold_in_cands": true,
 "pred_id": "P04", "probabilities": {"P04": 0.97, "P01": 0.01, "...": 0.0, "NONE": 0.0}, "confidence": 0.968,
 "pred_id_rev": "P04", "probabilities_rev": {"...": 0.0}, "confidence_rev": 0.95,
 "repeat": 0, "latency_ms": 840, "input_tokens": 2100, "output_tokens": 1}
```

- 앞 7칸은 `mentions.csv` 값 그대로 (`jev_normalize.run_one` 이 만드는 것과 같은 키)
- `cand_ids` = `[c["id"] for c in cands]` (NONE 제외, 순서 그대로) · `gold_in_cands` = `common.gold_in_candidates(m, cands)`
- `probabilities` 의 키 = criteria 키 전체(NONE 포함), 값 합 1
- 호출이 실패한 건은 지우지 말고 `{"...앞 칸...", "error": "<메시지>"}` 로 남긴다 — 채점기가 오류 수를 센다
- `config` 는 `k=all` 또는 `k=5` 처럼 Jev 와 같은 표기

## 5. 함정 — 미리 처리할 것

1. **`top_logprobs` 상한.** OpenAI 는 최대 20개로 알려져 있다(제3자 SDK 문서 기준, 공식 문서로 확인할 것). 20위 밖 라벨은 확률을 못 읽는다.
   가상 데이터는 프로젝트 16 + NONE = 17개라 괜찮지만, **실제 데이터에서 선택지가 20개를 넘으면** 확률이 비는 선택지가 생겨 확신도가 틀어진다.
   그때 쓸 방법을 정해서 이헌준과 맞춘다(예: 선택지별 예/아니오 확률을 따로 받아 정규화). **혼자 후보를 줄이지 않는다** — 후보가 달라지면 비교가 깨진다
2. **라벨 토큰.** A~Z 는 26개까지. 넘으면 두 자리 라벨이 여러 토큰으로 쪼개질 수 있다. 쓰는 토크나이저에서 라벨이 한 토큰인지 확인한다
3. **답 형식.** 첫 토큰에 공백 · 따옴표 · 괄호가 먼저 나오면 라벨 확률을 엉뚱한 위치에서 읽는다. 「라벨 한 글자만」 으로 강제하고, 첫 토큰이 라벨이 아니면 그 건을 `error` 로 남긴다
4. **추론(reasoning) 모델은 logprobs 를 안 줄 수 있다** — 모델을 고르기 전에 확인
5. **키 · 비용.** `.env` 에 키를 둔다(커밋 금지, `.gitignore` 에 있음). 전체 실행 전에 `--limit 20` 같은 소량 실행으로 형식을 먼저 확인
6. **정답 항목으로 돌린다.** 입력은 `mentions.csv` 의 `surface` 다. 추출 파트(가연 · 서현)의 결과를 넣지 않는다 — 추출 오류가 정규화 점수에 섞인다

## 6. 완료 기준

- [ ] `python3 check_parity.py out/jev_kall_….jsonl out/llm_kall_….jsonl` 이 「일치」 를 출력한다 (같은 mention, 같은 후보 순서, 같은 선택지 수)
- [ ] `python3 score.py out/jev_kall_….jsonl out/llm_kall_….jsonl --price-in <입력 $/100만 토큰> --price-out <출력 $/100만 토큰>` 이 오류 없이 비교표를 낸다
- [ ] `k=5` 로도 한 번 (Jev 결과 `out/jev_k5_….jsonl` 과 비교)
- [ ] 쓴 모델 ID · 단가 · 프롬프트 원문을 `RESULTS_llm_<MMDD>.md` 에 남긴다

## 7. 실행 뼈대 (참고용 — 그대로 써도 된다)

```python
import json, math, time
from common import load_data, build_candidates, gold_in_candidates
from jev_normalize import build_request

standard, mentions = load_data("data/sample")
for m in mentions:
    cands = build_candidates(m, standard, k=None)              # k 는 Jev 실행과 같게
    body = build_request(m, cands, model="")
    state = body["state"]
    q = body["questions"]["same_fwd"]
    ids = list(q["criteria"])                                    # 순서 그대로, 마지막이 NONE
    labels = [chr(ord("A") + i) for i in range(len(ids))]
    options = "\n".join(f"{lab}. {json.dumps(q['criteria'][i], ensure_ascii=False)}" for lab, i in zip(labels, ids))
    prompt = f"{q['instructions']}\n\n[state]\n{json.dumps(state, ensure_ascii=False)}\n\n[선택지]\n{options}\n\n선택지 라벨 한 글자만 답하라."
    # ... LLM 호출 (temperature=0, logprobs=True, top_logprobs=20, max_tokens=1)
    # top = {토큰: logprob} (첫 토큰 위치)
    # p = {i: math.exp(top.get(lab, -1e9)) for lab, i in zip(labels, ids)}; s = sum(p.values()); p = {k: v / s for k, v in p.items()}
    # pred = max(p, key=p.get); n = len(ids); conf = (p[pred] - 1 / n) / (1 - 1 / n)
```
