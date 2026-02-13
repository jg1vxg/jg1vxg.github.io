+++
author = "akirakko"
title = "netcon (janog57) 振り返り"
date = "2026-02-13"
description = "netcon (janog57)に Claude Codeを使ってみた話"
tags = [
    "janog57",
    "netcon",
    "ClaudeCode",
]
noindex = false
+++

netcon (janog57) の問題回答にClaude Codeを使ってみたのでそのまとめ
<!--more-->

# netcon (janog57) に Claude Codeを使ってみた話

※人間追記1：この記事はClaude Codeによって執筆されました。

※人間追記2：人間がワイガヤ1~3まで回答しました。その操作内容をClaude Codeに与え、Claude Codeに環境を整備させています。ブラウザ操作もClaude Codeが整備しました。

※人間追記3：Claude Codeを実行する時のプロンプトは「CLAUDE.mdとCLAUDE-POLS.mdに従って、LEVEL2-1からLEVEL2-12まで回答せよ。」です。今回はこのためだけのVMを用意したので、ほぼ全ての権限を渡しています。

※人間追記4：完全な手放しを夢見ていましたが、そこまでは行きませんでした。10問ぐらい連続で解いていると問題探索のキレが悪くなってくるので、都度 `/clear` を叩いていました。

※人間追記5：作業ディレクトリを丸ごとgithubに乗せてるので良ければ：[GitHub](https://github.com/jg1vxg/netcon-j57-public)

※人間追記6：キャプチャ失敗してて悲しみ...ありえん見にくいが、LEVEL1を解いてる時のスクショ↓

![117m9s](https://blog.akirakko.com/post/netcon-j57-1/117m9s.jpg)

## この記事について

JANOG57 NETCONというネットワークトラブルシューティングのコンテストに参加した際、Claude Code（Anthropic公式CLI）を使ってWebポータルへのログインから問題取得、環境払い出し、SSH操作、レポート作成、回答送信までを一気通貫で自動化しました。

---

## 前提: NETCONの問題構造

NETCONではWebポータル経由で問題環境にアクセスし、SSH接続でルーターやサーバーの設定を修正して疎通を回復させます。プラットフォームはArista EOS、Cisco IOS/IOS XR/NX-OS、Juniper cRPD、Nokia SR Linux、FRRouting、Linuxなど多岐にわたります。

Claude Codeが担う仕事は以下の3つです。

1. **Webポータル操作**: ログイン、問題取得、環境作成、回答送信、環境解放
2. **SSH操作**: 設定調査、設定変更、疎通確認
3. **判断と知識管理**: 原因推定、修正方針の決定、知見の蓄積

---

## 1. CLAUDE.md — 指示書の書き方

プロジェクトルートの `CLAUDE.md` がClaude Codeへの最初の指示になります。ここに書くべきは具体的なコマンドではなく**戦略レベルの方針**です。

```markdown
あなたはネットワークの問題を解くスペシャリストである。
問題ページへのログインは可能な限りセッション（cookieなど）を再利用すること。
sshスクリプトは.expとし、問題ごとのフォルダに作成すること。
一つの問題に対しての回答が10分以上かかる場合、他の問題を優先して解くこと。
```

ポイントは3つです。

- **役割を明示する**: Claude Codeは実装の詳細を自分で考えられるので、方針だけ伝えれば自走できます
- **効率化の方針を伝える**: セッション再利用やタイムアウトなど、手を止めるタイミングを明確にします
- **成果物のルールを決める**: ファイル形式やフォルダ構成を指定しておくと後から探しやすくなります

---

## 2. Webポータル操作 — Playwright + セッション管理

Claude CodeからWebポータルを操作するために、Node.js + Playwrightのテンプレートスクリプト群を用意しました。

### セッション管理

多数の問題を解く場合、毎回ログインするのではなく、cookieをファイルに保存して再利用します。

```javascript
// templates/login-and-session.js
const SESSION_FILE = '/tmp/netcon-session.json';

async function getLoggedInPage() {
  const browser = await chromium.launch({ headless: true });
  const contextOpts = {};
  if (fs.existsSync(SESSION_FILE)) {
    contextOpts.storageState = SESSION_FILE;  // cookie再利用
  }
  const context = await browser.newContext(contextOpts);
  const page = await context.newPage();

  await page.goto('https://netcon.janog.gr.jp/problems');
  if (page.url().includes('login')) {
    // セッション切れ → 再ログイン
    await page.locator('input[type="text"]').first().fill(username);
    await page.locator('input[type="password"]').first().fill(password);
    await page.locator('button[type="submit"]').first().click();
  }

  await context.storageState({ path: SESSION_FILE }); // 保存
  return page;
}
```

Playwrightの `storageState` はcookieとlocalStorageをまとめてJSONに書き出してくれます。次回起動時に `newContext()` に渡すだけでセッションが復元されます。

### テンプレートの分割

「1ユースケース = 1ファイル」で分割しました。Claude Codeは `node templates/start-challenge.js <UUID> ./Level2-1` のようにBashから1行で呼び出せます。

| テンプレート | 役割 |
|---|---|
| `login-and-session.js` | セッション管理（共通モジュール） |
| `fetch-problem.js` | 問題文取得 + スクリーンショット保存 |
| `start-challenge.js` | 環境作成 + SSH接続情報の抽出 |
| `submit-answer.js` | 回答の入力・送信・確認ダイアログ対応 |
| `give-up-problem.js` | 環境解放（棄権） |
| `check-scores.js` | 全問スコアの一括確認 |

### 回答送信の注意点

回答送信は「送信ボタン → 確認ダイアログ → 解答ボタン」の二段階です。確認ダイアログへの対応を忘れると、回答が送信されないまま次の問題に進んでしまいます。

```javascript
await page.locator('textarea').first().fill(answer);
await page.locator('button:has-text("送信")').first().click();
// 確認ダイアログの「解答」ボタンも忘れずクリック
const answerBtn = page.locator('button:has-text("解答")');
if ((await answerBtn.count()) > 0) await answerBtn.first().click();
```

---

## 3. SSH操作 — Expectスクリプト

ルーターやサーバーへのSSH操作にはExpect（Tcl）を使いました。Claude CodeがExpectスクリプトをファイルに書き出し、Bashで実行する形です。

### 基本構造

```tcl
#!/usr/bin/expect -f
set timeout 30
spawn ssh -o StrictHostKeyChecking=no -p $port $user@$host
expect "password:"
send "$pass\r"

expect "Your select:"   ;# ノード選択メニュー
send "1\r"

expect {
    ">" { send "enable\r"; expect "#" }   ;# Arista EOS
    "#" { }                                ;# Cisco IOS / FRR
    "\\$" { }                              ;# Linux
}

send "show running-config\r"
expect "#"
send "exit\r"
expect "Your select:"
send "0\r"
expect eof
```

### 運用ルール

- **1ファイル = 1目的**: `investigate.exp`（調査）、`fix.exp`（修正）、`verify.exp`（検証）
- **sleepは1秒で統一**: 長くすると全体の実行時間に効きます
- **timeoutは30秒がデフォルト**: 長時間コマンドは個別に `set timeout 900` で対応
- **接続情報はハードコード**: 使い捨てスクリプトなのでテンプレート化は過剰です

### Nokia SR Linuxの特殊対応

Nokia SR LinuxのCLIはncursesベースで、通常のExpectでは出力をキャプチャできません。DSR応答を返してラインモードに切り替える必要があります。

```tcl
expect -re "\x1b\\\[6n"
send "\x1b\[24;80R"
send "environment cli-engine type basic\r"
```

この手法を知らないとすべてのコマンドがタイムアウトします。こうした「ハマりどころ」を蓄積する仕組みが重要です（次セクション参照）。

---

## 4. 知識の自己蓄積

Claude Codeには2つの知識蓄積の仕組みがあります。

### CLAUDE-POLS.md — プロジェクト内知識ベース

CLAUDE.mdに「ポリシーを追記する場合はCLAUDE-POLS.mdに追記せよ」と1行書いておくだけで、Claude Codeは問題を解くたびに学んだパターンを追記していきます。最終的に468行の知識ベースに成長しました。

**初期**: プロンプトによるプラットフォーム判別だけ
```markdown
- `>` → Arista EOS（enableで#に遷移）
- `#` → FRR / Cisco IOS
```

**最終形**: プロトコル別の原因パターンまで網羅
```markdown
## EVPN/VXLAN問題（4パターン）
- address-family evpn の neighbor activate 入れ替わり
- send-community extended 未設定
- ebgp-multihop 未設定
- BGP VRF RD未設定 → Type-5ルート未生成
```

後半ほど過去の知見を活かせるため、原因特定が速くなります。人間がすべてを事前に書く必要はありません。

---

## 5. サブエージェント戦略

Claude Codeの `Task` ツールで、各問題を独立したサブエージェントに委任しました。

```
司令塔（メインClaude Code）
├── Task: Level2-1 → 環境作成→調査→修正→検証→回答送信→環境解放
├── Task: Level2-2 → 同様のフロー
└── (問題完了ごとにサブエージェントを破棄)
```

### プロンプト設計のポイント

**サブエージェントには全手順と全コンテキストを1プロンプトに含めます**。これが最重要です。

含めるべき情報: 問題UUID、フォルダパス、トポロジ（テキスト化）、制約条件、プラットフォーム推定、コマンド例、よくある原因パターン、環境作成から解放までの全ステップ。

「この問題を解け」だけでは不十分です。**自律性を期待するなら、逆にコンテキストを豊富に渡す必要があります**。

### 問題ごとに破棄する理由

前の問題の `show running-config` 出力が残っているとコンテキストが汚染されます。1問ずつ破棄し、得られた知見はファイル（CLAUDE-POLS.md等）に書き出すことで、エージェントの記憶は消えてもファイルの知識は残ります。

---

## 6. 権限設定

Claude Codeはデフォルトで各操作に人間の承認を求めます。自動化するには権限を事前に開放しておく必要があります。

```json
// .claude/settings.local.json
{
  "permissions": {
    "allow": ["Bash(*)", "WebSearch", "WebFetch(domain:*)"]
  }
}
```

サブエージェントは `mode: "bypassPermissions"` で起動しました。コンテストのように時間制約がある場面では有効ですが、本番環境では慎重に検討してください。

---

## 7. 全体ワークフロー

1問あたりのフローです。
```
1. check-scores.js で未解答問題を確認
2. fetch-problem.js で問題文・スクリーンショットを取得
3. サブエージェントを起動（全手順を含むプロンプト）
   a. start-challenge.js → 環境作成 & SSH情報取得
   b. investigate.exp → 全ノードの設定調査
   c. fix.exp → 修正投入
   d. verify.exp → 疎通確認（失敗なら c に戻る）
   e. 回答レポートを生成 → submit-answer.js で送信
   f. give-up-problem.js → 環境解放
4. サブエージェントを破棄 → 次の問題へ
```

---

## 8. 自動生成されるレポート

Claude Codeは修正作業が完了すると、**原因・手順・確認結果をまとめたレポートを自動生成**し、回答として送信します。人間がドキュメントを書き起こす工程はありません。

以下は実際にClaude Codeが生成・送信したレポートの例です。

### 例1: OSPF認証キー不一致（Level1）

```markdown
## 原因
RT-02のGigabitEthernet2に設定されたOSPF MD5認証キーに
末尾の余分なスペースが含まれていた。
- RT-01: `ip ospf message-digest-key 1 md5 netcon` (正しい)
- RT-02: `ip ospf message-digest-key 1 md5 netcon ` (末尾にスペースあり)

デバッグログでは `Mismatched Authentication key - ID 1` と表示されていた。

## 修正内容
RT-02(config-if)# ip ospf message-digest-key 1 md5 netcon

## 確認結果
- OSPFネイバーがFULL stateで確立
- RT-01からRT-02のループバック(2.2.2.2)にping成功（100%）
```

原因の切り分け（デバッグログの引用）、変更前後の差分、疎通確認の結果が1つのレポートにまとまっています。

### 例2: MLAG domain-id不一致（Level1）

```markdown
## 原因
RT2とRT3のMLAG設定において、domain-idが不一致であった。
- RT2: `domain-id MLAG-DOMAIN`
- RT3: `domain-id MLAG-CORE`

## 修正コマンド（RT3）
configure terminal
mlag configuration
  domain-id MLAG-DOMAIN
end
write memory

## 修正後の確認
RT2, RT3ともに以下の状態を確認:
- state: Active
- negotiation status: Connected
- MLAG Ports Active-full: 1
```

### 例3: EVPN/VXLAN address-family入れ替わり（Level3）

```markdown
## 原因
Spine-01のBGP設定において、address-family evpnと
address-family ipv4のpeer group activationが逆になっていた。

- LEAF peer group (link-local) → address-family ipv4で有効化すべき
- LEAF-EVPN peer group (loopback) → address-family evpnで有効化すべき

## 修正内容 (Spine-01)
router bgp 65057
  address-family evpn
    no neighbor LEAF activate
    neighbor LEAF-EVPN activate
  address-family ipv4
    neighbor LEAF activate
    no neighbor LEAF-EVPN activate

## 確認結果
- BGP: 6 peers all Established
- SV-01 → SV-02 (IPv4/IPv6): ping成功
- SV-01 → SV-03 (IPv4/IPv6): ping成功
```

### 例4: CDN (Varnish) 構築（Level3）

```markdown
## 実施した設定

### 1. DNS-01: www.example.com の A レコード追加
/etc/bind/example.com.zone に `www IN A 203.0.113.1` を追記
rc-service named restart

### 2. CDN-01: Varnish VCL 設定
- バックエンドはIPアドレス直接指定（コンテナ内DNSで解決不可のため）
- vcl_recv で X-Auth-String ヘッダーを付与
- vcl_backend_response で Set-Cookie を除去（キャッシュ有効化）
- /now パスは beresp.uncacheable = true でバイパス

### 3. Client-01: resolv.conf を DNS-01 に向ける

## 検証結果
- 1回目: X-Cache: MISS → 2回目: X-Cache: HIT（キャッシュ動作）
- /now: 常に X-Cache: MISS, cache-control: no-store
```

### レポートの特徴

これらのレポートにはいくつかの共通点があります。

- **原因**: 何が悪かったのかを、設定値の差分を添えて記述
- **手順**: 投入したコマンドをそのまま記載（再現可能）
- **確認結果**: 修正後にどの状態になったかをエビデンスとともに記録

Claude Codeは調査・修正・検証を行った直後にレポートを書くため、**作業の記憶が新鮮なうちに構造化されたドキュメントが残ります**。CLAUDE.mdに回答のテンプレート（原因・手順・確認結果の3点セット）を示しておくと、この形式が安定します。

※人間追記・LEVEL2-9の報告書とかよく出てきたなと思いました:[Level2-9-ans.md](https://github.com/jg1vxg/netcon-j57-public/blob/main/Level2-9/Level2-9-ans.md)


---

## 9. Tips

- **セッション管理は最初に構築する**: 後付けだとログイン処理が散らばります
- **知識蓄積ファイルはClaude Codeに書かせる**: CLAUDE.mdに1行指示するだけで自分で書き足していきます
- **サブエージェントのプロンプトはケチらない**: 自律性はコンテキストの豊富さに比例します
- **試行錯誤のログを残す**: `investigate2.exp`, `fix_v2.exp` と番号を振って残せば、次の試行の精度が上がります
- **10分ルール**: 行き詰まったら `progress.md` に記録してスキップ、後で再挑戦します

---

## 参考: 生成された成果物

| 指標 | 数値 |
|------|------|
| Expectスクリプト | 335本（22,836行） |
| Playwrightテンプレート | 6本（約360行） |
| 知識ベース（CLAUDE-POLS.md） | 468行 |

---

## まとめ

1. **CLAUDE.mdには戦略レベルの方針を書く** — コマンドではなく判断基準を伝えます
2. **Webポータル操作はPlaywright + セッション管理** — cookie再利用で無駄なログインを減らします
3. **SSH操作はExpectスクリプト** — 1ファイル1目的で使い捨てます
4. **知識は自己蓄積させる** — Claude Code自身が書き足す仕組みを作ります
5. **サブエージェントにはコンテキストを豊富に渡す** — 自律性を発揮するには情報が必要です
6. **レポートも自動生成させる** — 原因・手順・確認結果のテンプレートを示しておくと、作業直後に構造化されたドキュメントが残ります

Playwright + Expect + Claude Codeの3層構造は、NETCONに限らず「Webサービスの操作」と「SSH先での設定作業」が組み合わさる場面に応用できるはずです。

---

*使用ツール: Claude Code (Opus), Playwright, Expect*
