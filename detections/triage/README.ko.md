# ARTEX ホストトリアージ(triage)

日本語 · [English](README.md)

[`artex_host_triage.py`](artex_host_triage.py) は、ARTEX が実行された形跡が疑われる**ホスト 1 台上で直接**
実行する読み取り専用のトリアージ(triage)スクリプトです。このディレクトリのそれ以外の資料は、SIEM([Sigma](../sigma/))・ネットワーク
センサー([Suricata](../suricata/))・脅威インテリジェンスプラットフォーム([侵害指標](../indicators/))を運用する防御者向けの
ものです。このスクリプトはそれとは別の対応者、つまり SIEM なしで不審なホストのシェルの前に立ち、ローカルの状態だけで「ここで
ARTEX が動いたのか」を素早く根拠をもって答えなければならない人のためのものです。

このスクリプトは、ディレクトリ内の他の資料が持つフィンガープリントをそのまま点検し、さらに**[侵害指標
リスト](../indicators/artex_indicators.csv)が意図的に Sigma ルールなしで残した 3 つのホスト・DB 指標**まで
点検します。この 3 つの指標は、ログやネットワークで観測されないため、ホストで直接確認するしかないものです
(`server-listen-port`、`recording-proxy-endpoint`、`postgres-exploration-schema`)。

## 何を点検するのか

すべての点検項目は、このリポジトリのソースで確認した文字列やパスに基づいており、各検出結果には、対応する Sigma ルールや
侵害指標の行と同じ限界を併記します。

- **待ち受けポート**: `:8787`(管理 UI)と `127.0.0.1:8788`(記録プロキシ)を確認します。この 2 つは
  [`cmd/artex/main.go`](../../cmd/artex/main.go) の `--addr`・`--proxy` フラグの既定値です。実行中の
  ホストでは `ss`・`netstat`・`lsof` の出力をパースし、`--ports-from` で渡したファイルから読むこともできます。
- **記録プロキシのアーティファクト**: レコーダーが初回実行時に作成する中間者(MITM)CA ファイル
  `<データディレクトリ>/traffic/_ca/mitmproxy-ca-cert.pem` と、その隣の `_index/index.sqlite`・`_blobs/` を
  確認します([`traffic/traffic.go`](../../traffic/traffic.go)。データディレクトリの既定値は実行ファイル隣の
  `data/` です)。この CA は、行き来する HTTP(S) を復号して記録する中間者トラフィックレコーダーの信頼アンカーです
  (MITRE ATT&CK T1557)。
- **ログマーカー**: ログファイルから、エンリッチプローブの User-Agent `artex-enrich/1.0`
  ([`enrich/enrich.go`](../../enrich/enrich.go))、自己アップデート送信の User-Agent `artex-selfupdate`
  ([`selfupdate/github.go`](../../selfupdate/github.go))、プラットフォームガードの監査マーカー
  ([`guard/guard.go`](../../guard/guard.go))を探します。ガードマーカーは非 ASCII のフレーミングまで原文のまま
  保持しており、grep が実際に一致するようにしています。ログローテーションで `.gz`・`.bz2`・`.xz` に圧縮された過去のログも展開して
  あわせて検査するため、ホストのログ履歴まで網羅します。ただし Python 標準ライブラリにコーデックがない形式
  (`.zst`・`.lz4`)は検査せず、**スキップしたファイルとして報告**します。黙ってクリーンと扱うことはないので、そのような
  ファイルは先に展開するか、手作業で `grep` して別途確認してください。
- **PostgreSQL 探索スキーマ**: ARTEX ストアのデュアルグラフテーブル(`exploration_nodes`・`_edges`・`_anchors` と
  `assets`・`companies`・`activity`、そして `agent_prompts` シード)を確認します
  ([`db/schema.sql`](../../db/schema.sql))。DSN を渡すと `psql` で照会し、`psql` がなければ手作業で実行できる
  読み取り専用クエリをそのまま出力します。
- **実行中プロセスの環境変数インジェクション**: プロキシ変数(`HTTP_PROXY`・`HTTPS_PROXY`・`ALL_PROXY`)と、mitmproxy
  CA(`mitmproxy-ca-cert.pem`)を指すツールチェーンの CA 信頼変数(`SSL_CERT_FILE`・`CURL_CA_BUNDLE`・
  `REQUESTS_CA_BUNDLE`・`GIT_SSL_CAINFO`・`NODE_EXTRA_CA_CERTS`)を**両方**持つプロセスを探します。ARTEX は、
  生成するすべての worker ツールにまさにこれらの変数を注入します([`agent/worker.go`](../../agent/worker.go) の
  `proxyEnv`、[`agent/proxyenv_test.go`](../../agent/proxyenv_test.go) がアサート)。**変数名がソースにハードコード**
  されているため(値だけ変更可能)、このフィンガープリントはオペレーターがバイナリ名やポートを変更しても残り、待ち受けポート単独より
  特異的です。実行中の Linux ホストでは `/proc` を読み、オフライン・フォレンジックイメージでは `--proc-from` で
  キャプチャした環境変数ダンプを読みます。プロキシ・CA がそろっていれば高い重大度、mitmproxy CA 単独または ARTEX 既定のプロキシ
  エンドポイント(`127.0.0.1:8788`)単独であれば中程度の重大度で報告しますが、mitmproxy CA のない社内プロキシは
  手がかりとして挙げません。

検出結果は**トリアージのための手がかりであり、断定ではありません。** また、どの項目にも該当しなかったからといって安全という意味では
ありません。オペレーターはバイナリ名の変更、データディレクトリの移動、ポートの変更ができるためです。

## プラットフォームサポート

このスクリプトは純粋な Python 3(標準ライブラリのみ)なので、Python 3 が動く環境ならどこでも動作し、Linux(CI
セルフテスト)と macOS で実際に動かして確認しました。OS に依存する点検は 2 つで、どちらも失敗せず
きれいに縮退します。

- **実行中のポート点検**: `ss` → `netstat` → `lsof` の順に試し、出力が得られた最初のツールを使います。Linux では
  `ss`・`netstat` が使われ、`ss` がなく `netstat` が Linux 形式の `-ltnp` フラグを受け付けない macOS・BSD(この場合は
  出力なしで終了)では `lsof -nP -iTCP -sTCP:LISTEN` に移り、同じ方式でパースします。リアルタイムに走査する
  代わりに保存しておいた一覧を読むには `--ports-from` を使ってください。
- **実行中プロセスの環境変数点検**: `/proc` を読むため Linux でのみ動作します。`/proc` がないホスト
  (macOS・BSD)では、クリーンではなく**スキップ**と報告するので、Linux ホストでダンプを取り `--proc-from` で
  渡してください(使い方を参照)。

残りの点検(記録プロキシのアーティファクト・ログマーカー・PostgreSQL スキーマ)は、ファイルシステム・ログファイル・(DSN があれば)
`psql` を読むため、OS には依存しません。

## 使い方

```sh
# ホストを最初から最後まで点検します
detections/triage/artex_host_triage.py \
    --data-dir /opt/artex/data \
    --log /var/log/syslog --log-dir /var/log/artex \
    --pg-dsn "$ARTEX_PG_DSN"

# 機械可読形式で出力し、1 つでも該当すれば非 0 で終了します
detections/triage/artex_host_triage.py --data-dir /opt/artex/data --json --exit-code

# オフライン・フォレンジックイメージ: キャプチャしたプロセス環境変数ダンプを読みます
#   ホストでダンプを作成する方法:
#   for p in /proc/[0-9]*; do echo "# $p"; tr '\0' '\n' < "$p/environ"; echo; done > proc_env_dump.txt
detections/triage/artex_host_triage.py --proc-from proc_env_dump.txt

# 再現可能なフィクスチャのセルフテスト(ホストの状態には触れません)
detections/triage/artex_host_triage.py --self-test
```

このスクリプトは純粋な Python 3 標準ライブラリのみを使います。インストールは不要で、ネットワークを使わず、
`--self-test` が使う専用の一時ディレクトリ以外にはどこにも書き込みません。ホストの状態(開いているポート、データ
ディレクトリ、ログファイル、そして DSN を渡したときのみデータベース)を読み、見つけた内容を出力します。終了コードは
デフォルトで `0` です(ゲートではなくトリアージ用途です)。`--exit-code` を指定すると、指標が 1 つでも該当した場合に `1`
で終了します。

## どのように正直さを保つのか

`--self-test` は合成ホストを作ります。仕込んだ CA・インデックス・ブロブストアのあるデータディレクトリ、各マーカーを含む
ログ、ポート一覧、キャプチャしたプロセス環境変数ダンプを作成したうえで、すべての点検がその上で発火することをアサートし、続いて
クリーンなホスト・正常なログ・社内プロキシのプロセスでは検出結果が **0** 件であること(誤検知がないこと)をアサートします。このセルフテストは [`detections` CI
ワークフロー](../../.github/workflows/detections.yml)に接続されており、[`detections/tests/run-all.sh`](../tests/run-all.sh)
が再度実行します。そのため、どれかの点検が壊れたり、指標が grep するソース文字列からずれたりすると、マージゲートで
失敗します。動かして確かめられない検知は主張にすぎない、という原則に従っています。
