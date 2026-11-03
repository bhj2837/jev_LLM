# jev_LLM — LLM 정규화 (Jev 비교용)

엔티티 정규화(프로필 표기 → 표준 목록 ID 또는 NONE)를 LLM으로 풀어, 이헌준의 Jev 정규화와 **같은 입력·같은 채점기**로 비교한다.
규격은 [`LLM_HANDOFF.md`](LLM_HANDOFF.md)(이헌준 작성)를 따른다.

| 파일 | 내용 |
|---|---|
| `llm_normalize.py` | LLM 정규화 실행기. logprobs로 선택지 확률을 읽고 규격 jsonl을 `out/`에 쓴다 |
| `LLM_HANDOFF.md` | Jev 쪽이 정한 규격 (수정하지 않음) |
| `LLM_TO_JEV_NOTES.md` | LLM 쪽 → Jev 쪽 합의 요청 사항 (20개 초과 선택지 처리 등) |

## 실행

이헌준의 `common.py`, `jev_normalize.py`, `score.py`, `check_parity.py`, `data/` 를 **이 폴더에 함께** 두고 실행한다(그 파일들은 수정하지 않는다).

```bash
pip install openai tiktoken
cp .env.example .env          # OPENAI_API_KEY 채우기 (커밋 금지)
python3 llm_normalize.py --probe                                  # 모델이 logprobs/top_logprobs=20 을 주는지 확인
python3 llm_normalize.py --data data/sample --k all --limit 20 --rev
python3 llm_normalize.py --data data/sample --k all --rev
python3 llm_normalize.py --data data/sample --k 5 --rev
python3 check_parity.py out/jev_kall_*.jsonl out/llm_kall_*.jsonl
python3 score.py out/jev_kall_*.jsonl out/llm_kall_*.jsonl --price-in <$/1M> --price-out <$/1M>
```

## 상태

- 실제 데이터·API로는 아직 실행 전 (가짜 모듈로만 동작 확인)
- 모델은 `gpt-4.1-mini`로 잠정. `--probe` 결과로 확정
- 선택지 20개 초과 시 기본은 error, `--over20 logit_bias` 방식은 이헌준과 합의 후 사용
