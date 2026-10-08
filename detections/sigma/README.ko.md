# ARTEX 検知ルール (Sigma / ホスト・ログ・SIEM)

日本語 · [English](README.md)

このディレクトリは ARTEX 検知セットのうち、ホスト・ログ・SIEM 層を担当します。ここに収録した
[Sigma](https://sigmahq.io) ルールは、防御ガイド([韓国語](../../docs/defense-ko.md) ·
[English](../../docs/defense-en.md))4 節の疑似ルールをベンダー中立の形式で正式化したもので、それぞれの
SIEM・EDR のクエリ言語に変換して使います。すべての指標は推測ではなく、このリポジトリのソースで実際に確認した
文字列や挙動に基づいています。ネットワーク層は [`../suricata/`](../suricata/)にあり、ATT&CK レイヤーや
指標 CSV・MISP エクスポート、ホストトリアージスクリプトを含む検知セット全体は
[`../README.ko.md`](../README.ko.md)が索引化しています。すべてのルールは、自身が所有する、または書面による許可を得た
システムを守る**防御・検知の目的にのみ**使用してください。

## アトミック(atomic)ルール

1 つのルールが 1 つの観測可能な事実に対応します。個別に変換しても、ツリー全体の一部として変換しても
かまいません。

- **[`artex_enrich_user_agent.yml`](artex_enrich_user_agent.yml)**: *ARTEX Asset Enrichment Probe
  User-Agent*。アセットエンリッチ(`enrich/enrich.go`)が送信するインバウンドの `artex-enrich/1.0` User-Agent です。
  対象側で観測する補助指標です。`level: high`。
- **[`artex_selfupdate_egress.yml`](artex_selfupdate_egress.yml)**: *ARTEX Self-Update Egress
  User-Agent*。自己アップデートルーチン(`selfupdate/github.go`)が発するアウトバウンドの `artex-selfupdate`
  User-Agent です。ホスト・フォレンジックの egress 指標です。`level: medium`。
- **[`artex_guard_audit_framing.yml`](artex_guard_audit_framing.yml)**: *ARTEX Platform Guard
  Audit-Log Framing*。ツール呼び出しがブロックされたときに監査ログへ記録されるプラットフォームガードの統制マーカーです
  (`guard/guard.go`)。ホスト・フォレンジック指標です。`level: high`。
- **[`artex_recording_proxy_ca.yml`](artex_recording_proxy_ca.yml)**: *ARTEX Recording-Proxy MITM CA
  Certificate Artifact*。記録プロキシが `_ca/mitmproxy-ca-cert.pem` に配置して生成する MITM CA ファイルです
  (`traffic/traffic.go`)。ホスト・フォレンジックの成果物で、ファイル名自体は単独で動作する mitmproxy とも共有されるため、
  ハンティングの手がかり(hunting lead)として扱います。`level: medium`。
- **[`destructive_command_hunting.yml`](destructive_command_hunting.yml)**: *Destructive Command
  Execution (ARTEX Guard-List Hunting)*。ARTEX ガードの内蔵拒否リスト(`db/db.go` シード)を反映した破壊的な
  シェル・DB コマンドです。ARTEX 固有のシグネチャでは**なく**、汎用的なハンティングの手がかりです。`level: medium`。

## 相関(correlation)ルール (行動ベース) · [`correlation/`](correlation/)

静的な文字列は変更できますが、行動は隠すのがより困難です。これらの Sigma **相関**ルールは、防御ガイド
4.1~4.2 節と 4.4 節の行動ベース層を正式化したものです。各ルールは上記のアトミックルールの 1 つを `id` で参照するため、
相関ファイル 1 つではなく **`sigma/` ツリー全体を変換する必要があります**(参照が解決されるように。[Sigma テスト](../tests/sigma/)が
まさにこの依存関係をアサートします)。

- **[`correlation/artex_enrich_scan_velocity.yml`](correlation/artex_enrich_scan_velocity.yml)**:
  *Enrichment Scan Velocity*。1 つの送信元が短いウィンドウ内に `artex-enrich/1.0` プローブを大量に送る場合です
  (エンリッチは並行度 4 で、レート制限なしに動作します)。単発ルールが見逃す速度を捉えます。`event_count`、
  `level: high`。
- **[`correlation/artex_enrich_fanout.yml`](correlation/artex_enrich_fanout.yml)**: *Enrichment
  Fan-Out*。1 つの送信元がエンリッチ User-Agent を異なる複数のホストへ拡散する場合です。リクエスト量ではなく
  接触した異なるホスト数がシグナルであり、アセット一覧を機械速度で走査する幅を捉えます。`value_count`、
  `level: high`。
- **[`correlation/artex_guard_block_burst.yml`](correlation/artex_guard_block_burst.yml)**:
  *Guard-Block Burst*。1 つのホストでプラットフォームガードの統制マーカーが繰り返される場合です。マーカーを引用しただけの
  文書ではなく、実際に稼働中の ARTEX の実行が自身のガードに触れている状況を示します。`event_count`、
  `level: high`。
- **[`correlation/artex_guard_marker_then_destructive.yml`](correlation/artex_guard_marker_then_destructive.yml)**:
  *Guard Marker With Destructive Command*。ガードマーカーと破壊的コマンドが 1 つのホストで 1 つのウィンドウ内に共に
  現れる場合です(防御ガイド 4.2 節、多段階)。ARTEX 固有のマーカーを、本来は汎用的な破壊的コマンドのシグナルと
  組み合わせるため、特異度が上がります。`temporal`、`level: high`。

しきい値とウィンドウは保守的な既定値ですので、自身のベースラインに合わせて調整してください。純粋な Web 多段階のケース(列挙 →
プロービング → 認証)は、依然として環境ごとのベースラインルールが必要です。そのパターンは ARTEX 固有の User-Agent 1 つには
還元されないためです。出発点として使える汎用的な行動ベースのベースラインテンプレートは[防御ガイド 4.2 節](../../docs/defense-ko.md)に
あり、ARTEX のソースに根拠を置けないため、この検証済みツリーからは意図的に除外しています。

## 範囲と正直さ: デプロイ前にお読みください

- **静的指標は変更できます。** オペレーターが User-Agent を変更したり CA ファイルを削除したりできるため、アトミック
  指標がないからといって安全という意味では**ありません**。長持ちするシグナルは、`correlation/` ルールが基準としている
  行動です。1 つの送信元が偵察から列挙、プロービング、認証・インジェクション試行へと進み、応答に適応し、休まず
  動き続ける流れがそれです。
- **破壊的コマンドルールは汎用的なハンティングです。** ARTEX ガードの拒否リストを反映していますが、同じコマンドを正当な
  管理者も実行します。ヒットは手がかりとして扱い、自身の環境を許可リストで絞り込み、それだけで ARTEX
  と断定しないでください。
- **ポートとスキーマはネットワークではなくホストフォレンジックです。** サーバーの既定ポート `:8787` と記録プロキシ
  `127.0.0.1:8788`(`cmd/artex/main.go`)、そして PostgreSQL の探索グラフスキーマは、不審なホストで直接
  確認するほうが適しています。そのため、ノイズの多いルールの代わりに[指標 CSV](../indicators/)と
  [ホストトリアージスクリプト](../triage/)として提供しています。
- **`logsource` とフィールド名は一般値です。** ルールは一般的な `category`・`product` のログソースとフィールド名
  (`cs-user-agent`、`CommandLine`、`TargetFilename`)を使います。変換時にパイプライン(`-p`)で自身の
  製品スキーマにマッピングしてください。下記のバックエンドの説明を参照してください。

## 検証と変換

[sigma-cli](https://github.com/SigmaHQ/sigma-cli)(pySigma)で検証しました。リポジトリのルートで実行します。

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install sigma-cli

# 構造 + ベストプラクティス検証 (期待値: 0 errors, 0 issues)
sigma check detections/sigma/

# このルールセットの文書化されたベースラインで SigmaHQ の慣例全体を検査 (期待値: 0 issues)
pip install pySigma-validators-sigmahq
sigma check --validation-config detections/tests/sigma_lint/validators.yml detections/sigma/

# 相関ルールが参照するアトミックルールを解決できるようツリー全体を変換
sigma plugin install splunk
sigma convert -t splunk --without-pipeline detections/sigma/
```

バックエンドごとに相関ルールのサポートが異なるため、`-t` の選択が重要です。Splunk、Elasticsearch EQL、Grafana Loki は
ツリー全体を変換し、Elasticsearch Lucene、OpenSearch、Microsoft `kusto` バックエンドはアトミックルール
5 つだけを変換します(ウィンドウは製品側でネイティブに表現します)。バックエンド別の実測表と `--without-pipeline`・
`-p` フィールドマッピングの説明は [`../README.ko.md`](../README.ko.md)にあり、
[`../tests/sigma_backends/`](../tests/sigma_backends/)が再現します。

## テスト

[`../tests/`](../tests/)配下の再現可能な 4 つのスイートがこれらのルールを扱い、いずれも Docker さえあれば動作します。
[`sigma/`](../tests/sigma/)は検証とツリー全体のコンパイル、そして相関ルールが単独では変換に失敗することを
アサートし、[`sigma_match/`](../tests/sigma_match/)はルールが悪意のあるサンプルで実際に発火し、良性サンプルでは
沈黙することを確認し、[`sigma_backends/`](../tests/sigma_backends/)は 5 種のバックエンドでの移植性を、
[`sigma_lint/`](../tests/sigma_lint/)は SigmaHQ バリデーターのベースライン全体(0 issues)を確認します。
[`../tests/README.ko.md`](../tests/README.ko.md)を参照してください。

## 貢献

検知への貢献を歓迎します。新しいルールは、すべての指標を観測可能な事実に基づかせ、限界を `description` に
明記し、SigmaHQ バリデーターのベースラインをクリーンに通過し
(`sigma check --validation-config ../tests/sigma_lint/validators.yml .`)、攻撃の手引きと読める内容を
含まないようにしてください。[`../../CONTRIBUTING.md`](../../CONTRIBUTING.md)と
[`../suricata/`](../suricata/)のネットワーク層、そして [`../README.ko.md`](../README.ko.md)を
参照してください。
