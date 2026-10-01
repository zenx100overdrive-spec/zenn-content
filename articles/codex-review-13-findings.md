---
title: "Claude Codeが自分でCodexにレビューを頼んだら、「直した」13件のうち3件が直っていなかった"
emoji: "🔍"
type: "tech"
topics: ["claudecode", "codex", "claude", "個人開発"]
published: false
---

9月30日の 23:30〜23:48、Claude Code は自分が書いたコードを Codex にレビューさせ、13件の指摘を直し、Codex にもう一度確かめさせました。2周目の返答には「**直した13件のうち3件は直っていない、新しい問題が2件**」とありました。私はこの20分のループに、一度も触っていません。

## 前提：何が決めてあって、何がその場の判断だったか

- **事前に決めてあったこと**：その夜の23:24に、私は Claude Code にこう送っていました。「Codexが適しているDevelopment / Code Review / Implementation / Technical Verification等については、ClaudeがTerminalからCodex CLIを起動し、Human承認なしで任務を委託して構わない」。条件は、ChatGPT のアカウント認証で動き、API の従量課金を使わないこと。Skill・CLAUDE.md・フックでの自動化はしていません。
- **その場の判断**：23:30に「今日書いたコードを Codex に独立してレビューさせる」と決めたのは Claude Code です。私は直前（23:21と23:24）に API の予算と Codex の使い方の指示を送っていましたが、23:30〜23:48 の間は何も送っていません。レビューの運搬も「直して」の一言もありません。
- 公式の `/codex:adversarial-review` などのプラグインは使っていません。`codex exec` を直接呼んでいます。

## 1周目：Codex に渡した依頼（全文。パスと手元の場所の名前だけ伏せています）

````markdown
# Task for Codex (delegated by Claude, 72h autonomy experiment) — independent code review, READ ONLY

Repository: <repo>. Do not modify any file. Do not run network commands.

Review these recent commits for real bugs (not style): `git log --oneline -8` — especially
- ai_work/refresh.py (runs every 20 min from a systemd timer)
- ai_work/capabilities.py (status rules; live checks; site check with cache)
- ai_work/human_work.py, ai_work/codex_tasks.py, ai_work/claude_tasks.py
- local_v01/se/importer.py (new: Human messages delivered mid-turn as `attachment`/`queued_command`)
- local_v01/se/logic.py (message_kind v0.3), local_v01/se/surface.py (_render_map, _render_steward)

Focus on: crashes on missing/odd data, wrong counts, duplicate imports, anything that could corrupt or lose
SE data, anything that could leak secrets into the Obsidian view or evidence text, timer re-entrancy.
You may run `python3 -m pytest -q tests/test_local_v01.py` (offline).

Answer in this exact shape, max 25 findings, most severe first:
FINDING <n>: <file>:<line> — <one-sentence defect> — <concrete failing input/state> — <severity HIGH/MEDIUM/LOW>
Then one line: VERDICT: <SAFE_TO_KEEP_RUNNING | FIX_BEFORE_NEXT_RUN>
````

実行したのはこの1行です（ファイルを変えない読み取り専用モード）。

```bash
codex exec -s read-only -C <repo> -o task-001-result.md - < task-001-review.md
```

## 1周目の結果：13件と「FIX_BEFORE_NEXT_RUN」

約6分で返ってきました。`VERDICT` は、私たちが Codex に選ばせた**二択のラベル**です（リリースの可否を Codex が決めたわけではありません。このラベルを採用して直すことにしたのは Claude Code で、私はそのとき見ていません）。

返ってきた HIGH の1件を、そのまま載せます。

```text
FINDING 4: local_v01/se/surface.py:413 — One malformed or partly written JSONL line makes the renderer discard the entire decisions file — a concurrent append leaving an incomplete final line makes open Human decisions appear to be zero — HIGH
```

私に判断を求める項目を1ページに並べる画面が、ログの最後の1行が書きかけだと「判断待ち0件」と表示してしまう、という指摘です。人に返すべきことが、黙って消える種類のバグでした。

内訳は HIGH 5件・MEDIUM 6件・LOW 2件。HIGH のうち3件は「秘密情報を伏せる処理の抜け」で、対象は手元の画面とログです。あとで手元の1,166ファイルを秘密の値の形式で検索し、**実際に漏れていた値は0件**でした。

## Claude Code が直した

- 13件を1件ずつ確認し、**12件を修正**。1件は「すでに作業の依頼だけを数えている」とコードで確かめ、**修正不要**としました。
- 23:41 に修正をコミット。

## 2周目：「直ったか確かめて」

Claude Code は Codex に、**1周目の Codex 自身の出力ファイル（要約していない原文）**と、**Claude Code の対応表（どれを直したと主張しているか）**を読ませ、修正のコミットを1件ずつ照らし合わせるよう頼みました。完全に独立した目ではなく、「直した」という主張を見せたうえでの確認です。

23:46 に返ってきた返答の中身：

| 1周目の13件 | 2周目の判定 |
|---|---|
| 直っていた | 10件 |
| **直っていなかった** | **3件** |
| 修正で新しく入った問題 | 2件 |

直っていなかった3件は、どれも「直した」側の思い込みでした（伏せる処理を本文には入れたが説明文と要約には入れていなかった、識別子を短くしすぎて衝突が残っていた、決済リンクの判定が文字列の有無だけだった）。Claude Code はこの5件を直し、23:48 にコミットしました。

## わかったこと（この1回の範囲で）

- **「直しました」という申告は、13件中3件で外れていた。** この回は、確認を別のAIにさせたことで、その3件が返答に出てきた。同じAIに確認させた場合と比べたわけではないので、「別のAIだから見つかった」とまでは言えない。
- **人に返すべき情報が黙って消えるバグ**（判断待ち0件）が、人を介さないレビューで先に見つかった。
- 3周目は回していない。最後の5件の修正は、**まだ誰にも確かめられていない**。

## 前と比べて

以前は、AIの出力を別のAIに貼り付けてレビューさせるのを、私が手でやっていました。8月末〜9月中旬の ChatGPT の会話ログで、「AIの報告文の形をした私の入力」を機械的に数えると約150回（セッション数ではなく、貼り付けた回の数）。この夜の20分は、その運搬が0回でした。1晩分なので、毎回そうなるとはまだ言えません。

## 事実の箱

- 時間：23:30 依頼 → 23:36 1周目の返答 → 23:41 修正 → 23:46 2周目の返答 → 23:48 再修正。3周目なし。
- 費用：Codex は ChatGPT のプラン内。その夜、`codex login status` が「Logged in using ChatGPT」であることと、API キーの環境変数が無いことを確認していた。
- 人の関与：この20分は0回。事前に「Codex に任せてよい」という許可だけを出していた。
