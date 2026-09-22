# fizz-memory-l3-summarizer

Fizz 記憶系の **L3**(要約層)。あるユーザの発言が閾値(既定 8 件)を超えたら、
LLM でその人の per-user 要約を rolling 更新する。

| 層 | 部品 | 役割 |
|---|---|---|
| L1 | fizz-memory-l1-ring | per-user 直近発言のインメモリ ring |
| L2 | fizz-memory-l2-events | 全発言の永続化と検索 |
| **L3** | **fizz-memory-l3-summarizer**(これ) | 閾値超で per-user 要約を rolling 更新 |

L1/L2 が「生の発言」を持つのに対し、L3 は「その人はこういう人」という圧縮表現を
持つ。recall はこの summary + 直近わずかだけを brain に渡す(会話は前の話題を
引きずらない方針)。

## I/O

stdin に 1 行 1 コマンド(NDJSON):

```jsonc
{"op":"summarize","user_id":7,"prev":"猫好きの常連","threshold":8,"events":[{...MemoryEvent...}]}
```

`events` 件数が `threshold` 以上のときだけ LLM を呼ぶ:

- 要約した → `{"user_id":N,"summary":"..."}`
- 閾値未満 → `{"user_id":N,"skipped":true}`(LLM 呼び出しなし)

`prev` があれば rolling 更新、無ければ新規要約。

## 設定

- `FIZZ_LLM_MODEL` — モデル指定(既定 `anthropic:claude-haiku-4-5-20251001`)。
- API キーは [fizz-llm-client](https://github.com/aiviecast/fizz-llm-client) が
  provider ごとの env(`ANTHROPIC_API_KEY` 等)から読む。

## 開発

```sh
almide check src/main.almd
almide test                 # 純粋ロジック (閾値判定・プロンプト組み立て) を検証
almide build src/main.almd -o build/fizz-memory-l3-summarizer
```

ツールチェーン: [almide](https://github.com/almide/almide) v0.26.6+。

## 契約

[fizz-protocol](https://github.com/aiviecast/fizz-protocol) の `memory`
(`MemoryEvent`)+ [fizz-llm-client](https://github.com/aiviecast/fizz-llm-client)
に依存。
