---
title: "Claude CodeにCodexを起動させるのをやめた。2つのAIを並べた1日、17回の実行がすべて正常終了だった"
emoji: "🪢"
type: "tech"
topics: ["claudecode", "codex", "systemd", "linux", "個人開発"]
published: false
---

Claude Code と Codex を一緒に使うとき、よくある形は「親子」です。Claude Code が考え、必要になったら `codex exec` で Codex を呼び、結果を受け取って次へ進む。指揮役が一人いるほうが分かりやすい——そう思われがちです。

自分もずっとそうしていました。そして、やめました。

親子にした結果、**役割が混ざった**からです。2つを並べ直したところ、10/5 の1日で Claude 側が8回、Codex 側が10回動き、終わった17回はすべて正常終了でした。この記事は、その構成と1日の数字です。

## 親子にすると、何が混ざるのか

自分の運用では、Claude Code に戦略と実装、Codex に検証と修復を任せています。ところが Claude Code が Codex を起動する形にすると、次のことが起きました。

- Codex の「検証 FAIL」を、Claude Code が戦略そのものを捨てる理由として扱う
- 逆に、Claude Code 側の自己テストの PASS が、Codex の独立検証の代わりのように扱われる
- どちらが正しいか、という上下関係が生まれる

どちらのAIも、自分の領域では正しいことを言っています。問題は能力ではなく、**呼ぶ側が結果の意味まで決めてしまう構造**でした。

## やったこと：起動はOS、会話はファイル

変えたのは3点だけです。

1. **起動・停止・再起動は systemd に任せる。** AIがAIを起動しない
2. **受け渡しはファイルだけ。** 各AIに受信箱と送信箱のフォルダを1つずつ持たせ、間に置いた小さな振り分けスクリプトが運ぶ
3. **1件の依頼につき、CLIを1回だけヘッドレスで動かす。** 会話を常駐させない

### systemd のユニット

Claude 側の例です。Codex 側も同じ形で、実行するスクリプトの引数だけが違います。（パスは記事用に短くしています）

```ini:~/.config/systemd/user/agent-claude-main.service
[Unit]
Description=Claude Main supervisor (runs the CLI once per inbox item)
StartLimitIntervalSec=600
StartLimitBurst=5

[Service]
Type=simple
Environment=PATH=%h/.local/bin:/usr/local/bin:/usr/bin:/bin
ExecStart=/usr/bin/python3 %h/agents/agent_main.py claude
Restart=always
RestartSec=20
KillMode=control-group
TimeoutStopSec=660

[Install]
WantedBy=default.target
```

`loginctl enable-linger` を有効にしておけば、ログインしていなくても動きます。

### 監督スクリプトの中心

常駐するのはAIではなく、この数十行の Python です。AIを使わず、決まったことだけをします。

```python
def step(self):
    if self.backoff_until and now() < self.backoff_until:
        return self.beat("BLOCKED")          # 使用量上限・認証切れは待つ。再試行で回さない
    if self.runs_today() >= cap[self.ai]:
        return self.beat("BLOCKED")          # 1日の実行回数の上限
    p = self.next_packet()                   # 受信箱の、まだ処理していない一番古い1件
    if p is None:
        return self.beat("IDLE")
    self.run(p)                              # claude -p / codex exec を1回だけ。タイムアウト付き
    self.beat("IDLE")
```

実行コマンドはこうです。

```bash
# Claude 側（ヘッドレス、権限は auto モード）
claude -p --permission-mode auto "$(cat prompt_with_packet_path.md)"
# Codex 側（API キーを外し、ChatGPT サインインだけで動かす）
env -u OPENAI_API_KEY -u CODEX_API_KEY \
  codex exec -s workspace-write -C ~/agents/wm/codex - < prompt_with_packet_path.md
```

プロンプトには、それぞれの担当範囲と「相手を起動しない」「結果は自分の送信箱に JSON で書く」だけを入れています。

### 生きているかは、ファイル1つで分かる

監督スクリプトは15秒ごとに、自分の状態を1つの JSON に上書きします。

```json
{"agent": "codex", "heartbeat_at": "2026-10-05T10:04:42+00:00",
 "state": "ACTIVE", "task": "P-5c6dfadbe6fa… 営業時間10:00〜15:00の制限撤廃・周知と適用後検証",
 "pid": 82434, "blocked_reason": null, "runs_today": 6}
```

更新が90秒止まったら OFFLINE とみなします。人間の確認画面も、この JSON とログを読むだけです。

## 1日の数字（10/5）

| | Claude 側 | Codex 側 |
|---|---|---|
| 起動した回数 | 8 | 10（うち1回は集計時点で実行中） |
| 正常終了（exit 0） | 8 / 8 | 9 / 9 |
| 1回の所要時間 | 15〜465秒 | 45〜375秒 |
| 途中で落ちた実行 | 0 | 0 |

AI同士の受け渡し（振り分けスクリプトが運んだ件数）は次のとおりです。

| 方向 | 種類 | 件数 |
|---|---|---|
| Claude → Codex | 作業依頼 | 5 |
| Codex → Claude | 結果 | 4 |
| Claude → Codex | 結果 | 2 |

この往復の間、人間は依頼の中継をしていません。人間がしたのは、別の窓口から最初の方針を出すことだけでした。

## 落ちたときの扱い

- **監督スクリプトを `kill -9` した場合：** systemd が約20秒で起こし直しました。プロセスIDは変わり、状態は IDLE に戻りました（Claude 側・Codex 側の両方で確認）。
- **CLIの実行が途中で落ちた場合：** 同じ依頼を最大2回まで再実行します。3回目で止め、人間宛ての合図を残します。
- **使用量の上限・認証切れ：** BLOCKED にして待ちます。再試行のループは作りません。

## 持ち帰れるチェックリスト

複数のAIを並べるときに、自分が守っている5つです。

1. AIにAIを起動させない。起動は systemd などOS側の仕組みに任せる
2. 受け渡しはファイル（受信箱・送信箱）にして、内容は JSON にする
3. 1件の依頼 = CLIの実行1回。会話を常駐させない
4. 状態は「ファイル1つ＋90秒ルール」で外から見えるようにする
5. 片方の結果は、もう片方への「証拠」として渡す。結論の上書きにしない

## まだ分からないこと

- 親子構成と比べて、**仕事の質**が上がったかは測っていません。この記事の数字は、止まらずに回ったかどうかだけです
- 集計したのは1日分です。日をまたいだ安定性は、これから見ます
- Claude 側は Pro プランの使用量を対話用のセッションと共有しています。1日の上限を何回にすると対話が詰まるかは、まだ測っていません

親子をやめても、指揮役がいなくなるわけではありません。指揮しているのは、AIではなく、OSと数十行のスクリプトでした。
