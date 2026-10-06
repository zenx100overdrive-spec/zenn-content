---
title: "Claude Codeが今なにをしているか縦モニターに映したら、操作の79%はコードを書くことではなかった"
emoji: "🖥️"
type: "tech"
topics: ["claudecode", "codex", "可視化", "python", "linux"]
published: true
---

Claude Code に仕事を任せていると、画面の向こうで「AIがコードを書いている」と思いがちです。進捗を見る画面を作るなら、書いたファイルの数やコミットを並べればいい——そう考えていました。

実際に、作業の様子をそのまま縦モニターに映してみました。直近3時間のツール操作95件のうち、**ファイルを書く・直すのは10件、コマンドの実行が75件（79%）**でした。AIの仕事の大半は、コマンドを打って結果を確かめることだったのです。

この記事は、その観測画面の作り方です。新しいログを書かせる仕組みは足していません。**Claude Code が最初から残している会話記録を、読むだけ**です。

## 何を表示しているか

縦型（1080×1920）のモニター1枚に、上から次の5つを並べています。

| 領域 | 内容 |
|---|---|
| CURRENT | 今の目標・実験・作業（自分で書く小さな JSON） |
| LIVE CODE | 最後に触ったファイルの中身・差分・コマンドと出力。一番大きく取る |
| AI CONVERSATION | Claude Code と Codex が何を言っているか |
| ACTIVITY STREAM | 誰が・何を（READ / WRITE / EDIT / RUN / TEST）・どこに |
| SIGNAL | 異常があるときだけ赤く出す |

グラフや KPI は置いていません。数字より、**今どのファイルの何行目を書いているか**のほうが、見ていて状況が分かったからです。

## データの出どころ：Claude Code の会話記録

Claude Code は、セッションごとの会話を JSONL で保存しています。

```text
~/.claude/projects/<作業ディレクトリ名>/<session-id>.jsonl
~/.claude/projects/<作業ディレクトリ名>/<session-id>/subagents/*.jsonl   # サブエージェント
```

1行1レコードで、必要なのは次の2種類だけです。

```json
{"type": "assistant", "timestamp": "...", "entrypoint": "cli",
 "message": {"content": [
   {"type": "text", "text": "まず設定を確認します"},
   {"type": "tool_use", "id": "toolu_x", "name": "Edit",
    "input": {"file_path": "/path/app.py", "old_string": "...", "new_string": "..."}}]}}
{"type": "user", "timestamp": "...",
 "message": {"content": [{"type": "tool_result", "tool_use_id": "toolu_x", "content": "..."}]}}
```

- `text` は、AIの発言として会話欄へ
- `tool_use` は、操作として ACTIVITY STREAM へ。`Write` / `Edit` は中身や差分をそのまま LIVE CODE へ
- `Bash` は、対応する `tool_result` を `tool_use_id` で結び付けて、「コマンド＋出力」として表示

`entrypoint` が `sdk-cli` なら `claude -p` で動かしたヘッドレス実行、`cli` なら対話セッションです。これで、人と話している Claude と、裏で依頼を処理している Claude を分けて表示できます。

## 末尾だけ読む

会話記録は大きくなります。自分の環境では、いちばん大きいファイルが108.8MBありました。毎回全部を読むと重いので、**各ファイルの末尾約0.9MBと、直近3時間に更新されたファイルだけ**を読みます。

```python
def tail_lines(path, n=900_000):
    with open(path, "rb") as f:
        size = f.seek(0, 2)
        f.seek(max(0, size - n))
        lines = f.read().decode("utf-8", "replace").splitlines()
    return lines[1:] if size > n else lines   # 途中から読んだ最初の1行は捨てる
```

これで、10ファイル分を組み立てても1回0.15秒でした。画面は2秒ごとに取り直しています。

## 操作の種類を決める

`tool_use` の `name` から、表示用の動詞を決めています。

```python
def verb(tool, inp):
    if tool == "Write": return "WRITE"
    if tool in ("Edit", "MultiEdit"): return "EDIT"
    if tool == "Read": return "READ"
    if tool in ("Grep", "Glob"): return "SEARCH"
    if tool == "Bash":
        return "TEST" if re.search(r"\bpytest\b|\bnpm (run )?test\b", inp.get("command", "")) else "RUN"
    return tool.upper()
```

冒頭の79%は、この分類で数えたものです（直近3時間・各ファイルの末尾の範囲で、ツール操作95件）。

| 操作 | 件数 |
|---|---|
| RUN | 75 |
| WRITE | 5 |
| EDIT | 5 |
| READ | 3 |
| WEB | 3 |
| TEST | 2 |
| その他 | 2 |

## 秘密を映さない

会話記録には、コマンドの出力がそのまま入ります。画面に出す前に、次の2つを必ず通しています。

```python
SECRET = re.compile(r"(?i)(\b[0-9a-f]{40}\b|sk-[A-Za-z0-9_-]{8,}|bearer\s+[A-Za-z0-9._-]{12,}"
                    r"|(api[_-]?key|[a-z_]*token|[a-z_]*secret|password|private[_-]?key)\s*[\"']?\s*[:=]\s*[\"']?[^\s\"',]{6,})")
SENSITIVE_PATH = re.compile(r"(?i)(credential|\.env\b|/\.auth/|token|secret|password|id_rsa|\.ssh/|oauth|cookie)")
```

- 文字列は `SECRET` に当たる部分を `[masked]` に置き換える
- パスが `SENSITIVE_PATH` に当たるファイルは、中身を一切表示しない

実際に、トークンをファイルへ保存した操作がありましたが、画面には「表示しません」とだけ出ました。

## Codex も同じ画面に出す

Codex CLI も、`~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl` に記録を残しています。

- `response_item` の `message`（role が assistant）を会話欄へ
- `custom_tool_call` / `function_call` の入力からコマンドを取り出し、対応する `*_output` と結び付けて表示

1行目の `session_meta` に作業ディレクトリ（`cwd`）が入っているので、**仕事用のディレクトリで動いたセッションだけ**を表示しています。個人的に使っている別のプロジェクトの会話は、画面に出ません。

## 動きは、本物のイベントだけ

見ていて「動いている」と感じられるように、いくつか演出を入れています。ただし、どれも実際のイベントがあったときだけです。

- 最後の操作が60秒以内なら「LIVE」の印を点ける。それ以外は「n分前」と出す
- 新しい書き込みがあったときだけ、行を上から順にふわっと表示する
- 何も起きていないときは、何も動かさない

考えているふりのアニメーションは入れていません。止まっているなら、止まって見えるのが正しいと考えています。

## まだ分からないこと

- 79%という割合は、自分の3時間分の観測です。作業の種類（調査中心か実装中心か）で大きく変わるはずです
- 会話記録の形式は Claude Code の内部の形式です。バージョンが変わると、読み方の修正が要るかもしれません
- サブエージェントが同時に何体も動いたとき、どれを LIVE CODE に出すのが見やすいかは、まだ試していません

AIの作業を「見える化」するのに、AIへ報告を書かせる必要はありませんでした。もう残っている記録を、読むだけで足りました。
