# swallow 設計メモ v0.2

2026-11-24 · @Crux

## 概要

swallow は、Citlalpol が渡す接続を受けて、**プロトコルを知る必要のある仕事**を受け持つプログラムです。

- **すること**：設計者が `access.json` に書いた `stages`（何を送るか／何を送ったら何が返ってきてほしいか）で要求を照合し、書かれた
  応答を返し、書かれた名前のプログラムを発火させる（`swallow serve`）。書かれた応答を本物に当てて照合する（`swallow verify`）。
- **デフォルト**：設計者が何を返すか・何を発火させるかを決めていないところでは、swallow がデフォルトの受け手になります。届いたものを
  記録し（決めるための材料）、本物と取り違えようのない答え方をします（「決まっていないとき」の節）。
- **しないこと**：要求の意味を推測すること。読むのは人が書いたものだけです。外への接続の遮断と経路（それは Citlalpol）。
- **位置**：Xipe Totec にとっての embody と同じです。Citlalpol は swallow がなくても段階 1 の道具として完結し、swallow は Citlalpol の
  取り決め（起動・渡し方・環境変数）に従う一つのプログラムです。Pact の mock server と provider verification に当たりますが、
  アプリの外（ネットワークの層）から行うので、アプリには何も足しません。
- **状態**：素朴な版が動く（2026-11-24。S1・S2：HTTP/1.1、`send` / `expect`、決まっていないときの 501。Citlalpol の `test/run.sh`
  で確かめた）。まだ：`fire`、`verify`、`draft`、TLS、HTTP 以外。始まりの話は [idea.md](idea.md)。全体の設計は Citlalpol の `docs/ja/design-memo.md`。

## 実装（v0.2、素朴な版）

`bin/swallow`（python3、一つのファイル）、`swallow serve` だけ。

- 要求を一つ読む（頭は 64 KiB、本文は `Content-Length` の 1 MiB まで）。HTTP/1.1 の要求行でなければ、何も返さずに閉じて
  `captured`（`kind: not-http`）とだけ言う
- `send` を上から照合（`method` は一致か `*`、`path` は glob）。当たった項目の `do`：省略か `stand_in` → `expect` の `on` が同じで
  `reply` のあるもの、`refuse` → `403`・`X-Swallow: refused`、`pass` → `500`・`X-Swallow: misplaced`（pass は Citlalpol の決定）
- `reply` は書いたままの `{status, headers, body}` か、access.json のディレクトリからの JSON ファイルのパス。`body` が JSON の値なら
  `application/json`
- 決まっていないとき：`501`・`X-Swallow: undecided`
- 結果は `CITLALPOL_RESULT_FD` へ JSON 1 行：`request`（メソッド、パス（問い合わせは**名前**だけ）、ヘッダーの**名前**、本文の**大きさ**）、`matched`、`replied`、
  `default`、`captured`
- 結果の口は fd 3 ではなく、Citlalpol が `CITLALPOL_RESULT_FD` で番号を渡す形にした（未決だったものの答え）

## Citlalpol との分担

| | Citlalpol | swallow |
| --- | --- | --- |
| 段階 1（接続：名前、ポート、ClientHello） | 決める（遮断 / 通す / 代役へ） | — |
| 段階 2（何を送るか） | 経路だけ。中身を読まない | 要求を読み、`send` と照合する |
| 段階 3（何が返ってきてほしいか） | — | `expect` の `reply` を返し、`fire` を発火させる |
| 本物との照合 | `citl verify` で起動し、通信を記録する | `swallow verify` が照合する |
| 記録 | すべての試み（citld） | 照合の結果（`matched`、`fired`、`captured`）を citld へ返す |

swallow は Citlalpol から次を受け取ります（Citlalpol の「ローカルのプログラムを起動する」の節）。

- 接続：`stdio`（既定）か `fd`（socket activation）
- 環境変数：`CITLALPOL_PROTOCOL`、`CITLALPOL_HOST`、`CITLALPOL_PORT`、`CITLALPOL_SNI`、`CITLALPOL_RUN`、`CITLALPOL_LABEL`、
  `CITLALPOL_ACCESS`（その宛先の `stages` を書いた JSON のパス）

swallow から citld へは、1 件の要求ごとに JSON 1 行を返します（stdout とは別の口。Citlalpol が渡す `CITLALPOL_RESULT_FD`）。

```json
{"request":{"method":"POST","path":"/v1/charges","headers":["content-type","authorization"],"body_bytes":312},
 "matched":"charge","replied":"citlalpol/replies/charge.json","fired":"after_charge","default":false}
```

## swallow serve

```
swallow serve            （Citlalpol が起動する。stand_ins.json に {"exec": ["swallow", "serve"], "via": "stdio"}）
```

1. `CITLALPOL_ACCESS` の `stages` を読む
2. 接続からプロトコルに従って要求を読む（HTTP/1.1 から始める）
3. `send` の `match` と照合する。当たった項目の `do`：
   - `stand_in`：`expect` の中で `on` がその項目のものを探し、`reply` を返し、`fire` を発火させる
   - `refuse`：本物と取り違えようのない拒否を返す（下の「答え方」）
   - `pass`：swallow は本物へ出さない。`pass` は Citlalpol の段階 1 の決定なので、ここに来るのは設定の誤り。そう返して記録する
4. 当たらなければ「決まっていないとき」の扱い

### reply（応答の表）

リポジトリの中のデータで、設計者が書きます。叩き台（HTTP）：

```json
{ "status": 201,
  "headers": { "content-type": "application/json" },
  "body": { "id": "ch_test_0001", "status": "succeeded", "amount": "{request.body.amount}" } }
```

- `{request.…}` で要求の値を差し込めるようにするか（どこまで許すか）は未決。計算が要る応答は `fire` のプログラムに任せる

### fire（発火）

`expect` の `fire` に書かれた**名前**を、この機械の `stand_ins.json` で引いて起動します（起動するコマンドは commit されるファイルに
書かない、という Citlalpol の決まりのまま）。

- 発火させたプログラムには、要求（伏せるものは伏せた形）を stdin に JSON で渡す
- 応答の前か後か、発火したプログラムが応答を変えられるかは未決（「未決」）
- 何が発火したかは必ず記録に残す（`fired`）

## 決まっていないとき：swallow がデフォルト

設計者が「何を返すか」「何を発火させるか」を決めていないところでは、swallow が受け手になります。

| 決まっていないこと | swallow のすること |
| --- | --- |
| `send` に当たる項目がない | 要求を `captured` として記録し、「決まっていない」答えを返す |
| 項目はあるが `expect` に `reply` がない | 同上（`matched` には項目の id を残す） |
| `reply` はあるが `fire` がない | 応答だけ返す。発火はしない（`fired: null`）。これは「発火しない」と決めたのと同じに扱う |
| 宛先が `stand_in` だが `as` がない | Citlalpol が swallow を起動する（Citlalpol 側のデフォルト。下の節） |

### 答え方：本物と取り違えようのないこと

決まっていない要求に「それらしい」答えを返すと、アプリが成功したと思い込み、評価の結果が誤ります。swallow の答えは、必ず失敗として
読まれる形にします。

- HTTP：`501 Not Implemented`、ヘッダー `X-Swallow: undecided`、本文に「この要求はまだ決まっていない（記録の id）」
- HTTP 以外、または読めないもの：何も返さずに接続を閉じる（段階 1 の遮断と同じ見え方）

### 記録：決めるための材料

`captured` の要求は、設計者が `stages` を書くための材料です。

- 秘密は伏せる（`Authorization`、`Cookie` などの値、本文は大きさと種類だけ。人が明示したときだけ本文を残す）
- `swallow draft` で、`captured` から `send` と `expect` の下書きを作る（全部 `refuse`、`reply` は空。人が直す）
- 記録は citld が持つ（`~/.local/share/citlalpol/`）。swallow 自身はファイルを持たない

### TLS の中身が読めないとき

ローカルの CA がまだない（Citlalpol の M4 の前）か、対象が証明書を固定しているときは、swallow は TLS の中身を読めません。その場合は
段階 1 の記録（SNI・ALPN）だけを残して閉じます。デフォルトが swallow でも、読めないものは読めないと記録します。

## swallow verify

```
citl verify <name>      （citld が stand_ins.json の role: "verify" を namespace の外で起動する）
swallow verify          （そのとき起動されるもの）
```

- `expect` の各行の `verify` を見る：`safe`（副作用のないもの）だけを本物に当てる／`never` は当てない／`elsewhere` は書かれた検証用の宛先に当てる
- どこを比べるかは設計者が `expect` に書けるようにする（時刻や ID のように毎回変わるものがあるため）。書かれていなければ、状態コードと
  本文の形（キーと型）だけを比べる
- 結果は行ごとに 1 行の JSON：`{"row": "charge", "result": "match" | "differ" | "skipped", "why": "…"}`
- 本物との通信は Citlalpol を通るので、すべて記録に残る

## Pact との関係

- 段階 3 の表（`send` と `expect`、`reply`）を Pact の形式に書き出せるようにする（`swallow export pact`）。Pact の broker や provider
  の CI とつなげる道を残す
- 逆向き（Pact の契約から `stages` を作る）も、形が合えば可能

## 最初の受け入れ：ping の 4 段（Citlalpol と一緒に）

Citlalpol の設計メモ「最初の受け入れ：ping の 4 段」の c・d が swallow の受け持ちです。`citl-ping` は名前に繋いで `GET /ping` を送り、
`pong` を待つ小さなプログラムです。

| | 条件 | swallow に期待すること |
| --- | --- | --- |
| c | `access.json` に宛先（`pong.citlalpol.test`）を宣言したが、`expect` に `reply` がない | 決まっていない要求として `501`・`X-Swallow: undecided` を返し、`captured` を記録する（S1） |
| d | `expect` に `reply`（本文 `pong`）を書いた | `send` の `match`（`GET /ping`）に当て、`pong` を返し、`matched` を記録する（S2） |

a（Citlalpol の外）と b（宣言なしで遮断）は Citlalpol だけで決まり、swallow は起動されません。

## マイルストーン

| | 内容 | 終わったと言える条件 |
| --- | --- | --- |
| S0 | この設計 | 未決のうち S1 に要るものが決まる。Citlalpol の M2（ローカルのプログラムの起動）の取り決めと合う |
| S1 | `serve` のデフォルトだけ（HTTP/1.1）：記録して 501 を返す | ping の c が通る。Citlalpol の下で、宣言のない HTTP の要求が `captured` として記録され、アプリには 501 と `X-Swallow: undecided` が返る。`Authorization` の値は残らない |
| S2 | `serve` の照合と応答：`send` の `match`、`expect` の `reply`、`swallow draft` | ping の d が通る。書かれた `reply` が返り、`matched` が記録に残る。`captured` から作った下書きを直して入れると、次の評価でその要求が先へ進む |
| S3 | `fire` | 書かれた名前のプログラムが発火し、`fired` が記録に残る |
| S4 | `verify` | `safe` の行だけを本物に当て、結果が行ごとに返る |
| S5 | TLS（Citlalpol の M4 のローカルの CA と組む）、HTTP 以外、`export pact` | |

## 未決

- 書く言語（Citlalpol と揃えるか）
- `reply` に要求の値を差し込めるようにするか、どこまで
- `fire` は応答の前か後か、発火したプログラムが応答を変えられるか
- 決まっていない要求への答え：HTTP の `501` でよいか（`503` や、アプリの種類によって変えるか）
- `match` に何を書けるか（パスの型、クエリ、ヘッダー、本文の一部）
- ライセンス
