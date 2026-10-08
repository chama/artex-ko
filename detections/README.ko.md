# ARTEX 検知ルール

日本語 · [English](README.md)

> このディレクトリは、[防御・検知ガイド(../docs/defense-ko.md)](../docs/defense-ko.md) 4 節「検知ルール」の
> 疑似ルールを、それぞれの SIEM・EDR のクエリ言語に変換してすぐにデプロイできるベンダー中立の
> [Sigma](https://sigmahq.io) 形式に移したものです。ここに掲載したすべての指標は、**推測ではなく**、この
> リポジトリのソースで実際に確認した文字列や挙動に基づいています。すべてのルールは、自身が所有する、または書面による
> 許可を得たシステムを守る**防御・検知の目的にのみ**使用してください。

## アトミック(atomic)ルール

- **`sigma/artex_enrich_user_agent.yml`**: ARTEX のアセットエンリッチ(`enrich/enrich.go`)が送信するインバウンドの
  `artex-enrich/1.0` User-Agent です。対象側で観測する補助指標です。`level: high`。
- **`sigma/artex_selfupdate_egress.yml`**: 自己アップデートルーチン(`selfupdate/github.go`)が発する
  アウトバウンドの `artex-selfupdate` User-Agent です。ホスト・フォレンジック視点の送信(egress)指標です。
  `level: medium`。
- **`sigma/artex_guard_audit_framing.yml`**: ツール呼び出しがブロックされたときに監査ログへ記録されるプラットフォーム
  ガードの統制マーカー(`guard/guard.go`)です。ホスト・フォレンジック指標です。`level: high`。
- **`sigma/destructive_command_hunting.yml`**: ARTEX ガードの既定のブロックリスト(`db/db.go` シード)を
  そのまま反映した破壊的なシェル・DB コマンドです。ARTEX 固有のシグネチャではなく、汎用的なハンティングの手がかりです。
  `level: medium`。
- **`sigma/artex_recording_proxy_ca.yml`**: 記録用プロキシが `_ca/mitmproxy-ca-cert.pem` に配置して
  生成する MITM CA 証明書ファイル(`traffic/traffic.go`)です。ホスト・フォレンジックのアーティファクトで、ファイル名
  自体は単独で動作する mitmproxy と共有されるため、ハンティングの手がかりとして扱います。`level: medium`。

## 相関(correlation)ルール: 行動ベース

静的な文字列は変更できますが、行動は隠すのがより困難です。[`sigma/correlation/`](sigma/correlation/)
の Sigma **相関**ルールは、防御ガイド(4.1~4.2 節、4.4 節)の行動ベース層を収めています。各相関ルールは
上記のアトミックルールを `id` で参照するため、参照を解決するには単一の相関ファイルではなく `sigma/` ツリー全体を
変換する必要があります(下記参照)。

- **`sigma/correlation/artex_enrich_scan_velocity.yml`**: 1 つの送信元が短い時間ウィンドウ内に大量に送る
  `artex-enrich/1.0` プローブの塊です(エンリッチは並行度 4 でレート制限なしに動作します)。単発ルールが
  見逃す速度を捉えます。`event_count`、`level: high`。
- **`sigma/correlation/artex_enrich_fanout.yml`**: 1 つの送信元がエンリッチ User-Agent を複数の**異なる**
  ホストに運ぶ場合です。アセット一覧全体へ機械速度で広がる様子で、量(volume)だけでなく
  幅(breadth)が手がかりです。`value_count`、`level: high`。
- **`sigma/correlation/artex_guard_block_burst.yml`**: 1 つのホストでプラットフォームガードの統制マーカーが繰り返し
  記録される場合です。単にマーカーを引用した文書ではなく、稼働中の ARTEX の実行が自身のガードに触れて
  いるというシグナルです。`event_count`、`level: high`。
- **`sigma/correlation/artex_guard_marker_then_destructive.yml`**: 1 つのホストで時間ウィンドウ内にガード
  マーカーと破壊的コマンドが共に現れる場合です(防御ガイド §4.2、多段階)。ARTEX 固有のマーカーを、それ
  自体は一般的な破壊コマンドのシグナルと組み合わせて特異度を高めます。`temporal`、`level: high`。

しきい値と時間ウィンドウは保守的な既定値です。それぞれのベースライン(baseline)に合わせて調整してください。§4.2 の純粋な
Web 多段階のケース(列挙 → プローブ → 認証)は、そのパターンが単一の ARTEX 固有 User-Agent に還元されないため、
依然として環境ごとのベースラインルールが別途必要です。その出発点として使える汎用的な行動ベースの Sigma ベース
テンプレートを[防御ガイド §4.2](../docs/defense-ko.md#42-siem-相関ルール)に置きました。ARTEX のソースで根拠を
固定できないため、ここでテストされるルールツリーには含めていません。

## ネットワークルール (Suricata)

Sigma はホストとログのテレメトリを扱います。ネットワーク上で観測される ARTEX 固有の User-Agent は 2 つ
あり、どちらも [`suricata/`](suricata/)に [Suricata](https://suricata.io) ルールとして収められています。エンリッチ
プローバーの `artex-enrich/1.0`(`enrich/enrich.go`)には存在シグネチャ 1 つと高速列挙の変種 1 つ(sid
1000001・1000002)が、norma SDK の WebFetch ツールが攻撃段階で送る `norma/0.4`(`github.com/Autumn-27/norma/tool/webfetch.go`)には
存在シグネチャ 1 つ(sid 1000003)が対応します。それ以外の worker ツール(Bash で実行する `curl`・`nmap` など)は
独自の User-Agent を使うため ARTEX 固有のフィンガープリントがなく、ネットワーク層は意図的にこの 2 つの UA に絞って
います。範囲と TLS の注意点、`suricata -T` と参照 pcap による検証方法は
[`suricata/README.ko.md`](suricata/README.ko.md)を参照してください。

## ATT&CK カバレッジ

これらのルールがタグ付けする技法は、[MITRE ATT&CK](https://attack.mitre.org/) Navigator レイヤー
[`attack/artex_navigator_layer.json`](attack/)にまとめました。6 つの戦術(偵察、コマンド&コントロール、実行、インパクト、
認証情報アクセス、収集)にまたがる 8 つの技法で、各技法はルールの `attack.*` タグに基づき、検知強度(ARTEX 固有のシグネチャか、
汎用的なハンティングの手がかりか)でスコアを付けました。[ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/)
で開くと、どの ARTEX 行動をどのルールがカバーしているかが分かります。スコアの算出と技法↔ルールの対応、そして
正直な範囲(カバレッジは網羅性ではありません)は [`attack/README.ko.md`](attack/README.ko.md)を参照してください。
[整合性テスト](tests/attack/run.sh)がレイヤーとルールセットの食い違いを防ぎます。

## 侵害指標リスト (機械可読)

検知ロジックではなくアトミックな指標そのものを求める防御者のために、
[`indicators/artex_indicators.csv`](indicators/)は ARTEX が発する固有のフィンガープリントを CSV 1 ファイルに
まとめました。脅威インテリジェンスプラットフォームや SIEM のルックアップテーブル、ホストのトリアージ(triage)チェックリストにそのまま
投入できるよう、エンリッチ・自己アップデートの User-Agent、ガード監査マーカー、サーバー・プロキシの既定エンドポイント、記録
プロキシの CA 証明書、そして PostgreSQL 探索グラフスキーマのフィンガープリントを収め、各行には根拠となったソース
ファイルと(あれば)その上に構築したルールを併記しました。同じ指標をそのまま
インポートできる [MISP](https://www.misp-project.org/) イベント
([`indicators/artex_indicators.misp.json`](indicators/))としても提供しているため、MISP を使う、あるいはそこから
STIX にエクスポートする防御者は CSV の列を手作業でマッピングする必要がありません。ルールに基づくフィンガープリントは `to_ids`
でマークし、ホストフォレンジック用のポートとスキーマのフィンガープリントはマークしていません。汎用的なハンティングの手がかり(破壊コマンド)と、
norma SDK が共有する `norma/0.4` WebFetch User-Agent(Suricata sid 1000003 が捉えるネットワーク署名であり、
ARTEX 固有の文字列ではありません)は、誤検知を避けるためインポート用リストから意図的に除外しました。列構成、MISP タイプのマッピング、正直な注意点、そして CSV と
MISP イベントの食い違いを防ぐ整合性テストは [`indicators/README.ko.md`](indicators/README.ko.md)を
参照してください。

## ホストトリアージ(triage)

上記のルールは、SIEM・ネットワークセンサー・脅威インテリジェンスプラットフォームを使う防御者向けのものです。それとは別の対応者、つまり
SIEM なしで不審なホスト 1 台のシェルの前に立つ人のために [`triage/artex_host_triage.py`](triage/)を用意しています。ローカルの
状態だけで「ここで ARTEX が動いたのか」に答える読み取り専用スクリプトです。同じフィンガープリントを点検し、さらに
**CSV が意図的に Sigma ルールなしで残した 3 つのホスト・DB 指標**(サーバーの待ち受けポート、記録プロキシのエンドポイント、
PostgreSQL 探索スキーマ)まで点検します。この 3 つは、ログやネットワークで観測されないため、ホストで直接確認するしか
ありません。また、レコーダーが子プロセスに注入する環境変数の痕跡、つまり実行中のプロセスがプロキシ変数と
mitmproxy CA 信頼変数を併せ持つか(`agent/worker.go`)を、`/proc` や `--proc-from` ダンプから確認します。
各検出結果は、該当する侵害指標の行と同じ限界を持つトリアージの手がかりです。詳細は
[`triage/README.ko.md`](triage/README.ko.md)にあり、内蔵の `--self-test` が下記のマージゲートで実行されます。

## テスト

ルールには、Docker さえあれば実行できる再現テストが [`tests/`](tests/)に同梱されています。

- **Suricata** ([`tests/suricata/run.sh`](tests/suricata/run.sh)): まずルールファイル全体が
  `suricata -T --init-errors-fatal` でロードされることを検証し(どのキャプチャにも触れないルールであっても
  パース失敗は検出されます)、scapy で決定的なキャプチャを合成して `suricata -r` でその上を実行し、存在ルールが
  プローブごとに 1 回発火し、速度ルールはしきい値を超えると検知し、良性(benign)な User-Agent のキャプチャでは
  アラートが 0 であることをアサートします。バイナリのキャプチャはコミットせず、実行のたびに再生成します。
- **Sigma** ([`tests/sigma/run.sh`](tests/sigma/run.sh)): 下記「検証と変換」の `sigma check` と
  `sigma convert` の検証を実行可能なテストとして動かします。エラー 0 と、ツリー全体がバックエンドクエリに
  コンパイルされること、各アトミック指標の文字列がそのクエリまで残ること、そして相関ルールが単独では変換に失敗することを
  アサートします。最後のアサーションは、相関ルールが参照するアトミックルールに実際に依存していることを証明します。
- **Sigma リアルタイムイベントマッチング** ([`tests/sigma_match/run.sh`](tests/sigma_match/run.sh)): 上記の Sigma テストが
  ルールの有効性とコンパイルを証明するのに対し、このテストはアトミックルールと相関ルールが実際に発火することを証明します。
  各アトミックルールについて、代表的な悪意のあるサンプルイベントがルールを発火させ、正常なサンプルイベントは発火させないことを
  アサートします(例: `.mitmproxy/` 配下の単独 CA ファイルは、`_ca/` ディレクトリまで併せて要求する記録用プロキシルールを
  発火させません)。各相関ルールについては、しきい値を時間ウィンドウ内に 1 グループが満たす陽性タイムラインで
  発火し、しきい値未満・ウィンドウ超過・グループ分割・レッグ欠落のタイムラインでは沈黙することをアサートします。パースはすべて pySigma に
  任せ、テストはコンパイル済みの条件ツリーと集計仕様だけを辿り、相関ルールがどのイベントを取り込むかは、アトミックルールと
  同じマッチャーで判定します。「動かして確かめられない検知ルールは主張にすぎない」という原則を、Suricata と同様に Sigma 側にも
  適用します。
- **ATT&CK レイヤー** ([`tests/attack/run.sh`](tests/attack/run.sh)): ATT&CK カバレッジレイヤーが
  ルールと整合して維持されているかを確認します。スコアを付けた技法・戦術はルールセットの `attack.*`
  タグと正確に一致しなければならず、各技法は実在するルールファイルを指していなければなりません。レイヤーを更新せずにルールを
  追加すると(またはその逆だと)、テストは失敗します。
- **指標の根拠(source-of-truth)** ([`tests/indicators/run.sh`](tests/indicators/run.sh)): 各ルールが
  固定した指標が、今もアップストリームのソースが発するまさにその文字列であるかを確認します。`enrich/enrich.go` の
  `artex-enrich/1.0`、`selfupdate/` の `artex-selfupdate`、`guard/guard.go` のガードマーカー、`db/db.go`
  の破壊トークンが、ルールにも今なお固定されているかを見ます。他の 3 つのテストが見逃すドリフト、すなわちすべての
  ルールがコンパイルされ発火している最中に、アップストリームの再同期が User-Agent やマーカーを変えてしまう場合を
  捉えます。同じテストが機械可読な [`indicators/artex_indicators.csv`](indicators/artex_indicators.csv)
  を再読み込みし、公開されたすべての行が今もソースとルールに基づいていることをアサートするため、防御者がインポートする成果物も
  陳腐化しません。最後に、読み込むすべてのアップストリームソースが CI ワークフローの `push`・`pull_request` パスフィルターに
  含まれていることをアサートし、新たに固定したソース 1 つだけに触れた PR がテストをスキップして、そのドリフトがマージ
  ゲートを通過することがないようにします。これにより「推測ではなく、このリポジトリのソースで確認した文字列に
  基づく」(上記)という約束が、言葉ではなくガードになります。
- **MISP エクスポート整合性** ([`tests/misp/run.sh`](tests/misp/run.sh)): MISP イベント
  ([`indicators/artex_indicators.misp.json`](indicators/artex_indicators.misp.json))が有効な MISP
  ドキュメントであることを証明します。[pymisp](https://github.com/MISP/PyMISP) でロードされますが、pymisp のオブジェクトモデルは
  実在しない属性タイプを拒否するため、この成果物は MISP のように見えるだけでなく実際に
  インポートできます。また、上記の CSV と行単位で同期していることをアサートします。同じ値、指標ごとに意図した MISP
  タイプ・カテゴリ、そして CSV の正直さをそのまま反映する `to_ids`・`disable_correlation` フラグが
  一致しなければなりません(ルールに基づく = 対処可能なので `to_ids` オン、ホストフォレンジック用のポート = トリアージのヒントなので
  `to_ids` オフかつ相関無効)。このイベントは CSV と並べて手作業で管理されます。CSV にない説明
  注釈・UUID・タグを持つため、CSV の行を追加・削除したりタイプを変更したりする際は同じコミットで MISP イベントも
  修正する必要があり、両者が一致するまでこのテストは失敗します。
- **Sigma バックエンド移植性** ([`tests/sigma_backends/run.sh`](tests/sigma_backends/run.sh)): ルールが
  Splunk の例 1 つを超えて変換されることを証明します。ツリー全体(アトミック + 相関)が Splunk、Elasticsearch `eql`
  ターゲット、Grafana Loki にコンパイルされ、5 つのアトミックルールは Sigma 相関をサポートしないバックエンド(Elasticsearch
  `lucene`、Microsoft `kusto` バックエンド)でも引き続きコンパイルされます。下記「検証と変換」のバックエンド別サポート
  表を、再実行できる点検で裏付けます。
- **SigmaHQ 慣例リント** ([`tests/sigma_lint/run.sh`](tests/sigma_lint/run.sh)): SigmaHQ バリデーター
  全体(`pySigma-validators-sigmahq` プラグインによるもので、通常の `sigma check` はロードしません)を
  [`tests/sigma_lint/validators.yml`](tests/sigma_lint/validators.yml)に文書化したベースラインに合わせて
  実行し、イシュー 0 をアサートします。またバリデーター全体が実際に動作し、意図的に除外した文書化済みの 4 つの
  検査だけが残っていることを確認するため、ルールが新たな慣例違反(大文字小文字が誤ったタイトル、分類体系を外れた
  フィールド)を 1 つでも持ち込むとビルドが失敗します。

8 つのルールテストとは別に、ルールではない 2 つのゲートが同じ CI ワークフローと [`tests/run-all.sh`](tests/run-all.sh)
で一緒に動きます。1 つはハーネス同期検査(run-all.sh・CI・スイートディレクトリが同じスイートを同じ順序で呼び出しているかを
確認)で、もう 1 つは[ホストトリアージツール](triage/)の `--self-test`([`tests/triage-selftest.sh`](tests/triage-selftest.sh))
であり、合成ホストを作ってすべてのトリアージ点検が発火するか、クリーンなホストでは検出結果が 0 件であるかをアサートします。

各スクリプトは、アサーションが 1 つでも失敗すると 0 以外のコードで終了します。[`tests/README.ko.md`](tests/README.ko.md)
を参照してください。

## これらのルールを正直に読む方法

- **静的指標は変更できます。** オペレーターが User-Agent を別の値に設定できるため、
  `artex-enrich/1.0` や `artex-selfupdate` が**ないからといって安全という意味ではありません。** 長持ちする
  シグナルは*行動*です。1 つの送信元が偵察 → 列挙 → プローブ → 認証・インジェクション試行と続き、応答に適応し、
  休みなく動き続ける様子です。その層は防御ガイド(1 節・2 節・4.1~4.2 節)で説明し、上記の
  `sigma/correlation/` ルールがデプロイ可能な相関(速度、ファンアウト、ガードブロックのバースト、そしてガード
  マーカー+破壊コマンドの多段階)として収めました。純粋な Web 多段階のケースは、依然として環境ごとのベースラインルールが必要です。
- **破壊コマンドルールは汎用的なハンティングです。** ARTEX ガードのブロックリストを反映していますが、同じコマンドは正当な
  管理者も実行します。ヒットは手がかりとして扱い、環境に合わせて許可リストを設け、それだけで ARTEX と
  断定しないでください。
- **ポート指標は Sigma ではなくホストフォレンジック用です。** ARTEX サーバーの既定 `:8787` と記録プロキシ
  `127.0.0.1:8788`(`cmd/artex/main.go`)は、不審なホストで `ss`・`netstat` で確認するほうが
  適しています。そのためノイズの多いネットワークルールとしては掲載せず、防御ガイドに文書化し、トリアージ用に
  [指標 CSV](indicators/)に載せました。[ホストトリアージスクリプト](triage/)は、まさにこうしたホストローカルの点検(ポート、
  記録プロキシのアーティファクト、ログマーカー、PostgreSQL スキーマ)を、シェルアクセスはあるが SIEM がない対応者のために
  代わりに実行します。

## 検証と変換

これらのルールは [sigma-cli](https://github.com/SigmaHQ/sigma-cli)(pySigma)で検証しました。再現するには:

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install sigma-cli

# 構造 + ベストプラクティス検証 (期待値: エラー 0、イシュー 0)
sigma check detections/sigma/

# この独立したルールセット向けの文書化されたベースラインで SigmaHQ 慣例全体を検査 (期待値: イシュー 0)。
# 上記の通常の `sigma check` は、これらのバリデーターをロードしません。
pip install pySigma-validators-sigmahq
sigma check --validation-config detections/tests/sigma_lint/validators.yml detections/sigma/

# 対象クエリ言語にコンパイル、例: Splunk
sigma plugin install splunk
sigma convert -t splunk --without-pipeline detections/sigma/artex_enrich_user_agent.yml

# 相関ルールが id で参照するアトミックルールを解決できるようツリー全体を変換
sigma convert -t splunk --without-pipeline detections/sigma/
```

ベースラインは SigmaHQ の慣例をすべて強制しますが、SigmaHQ モノレポのファイル整理体系と分類体系を含む 4 つの
検査だけは、この独立したルールセットには該当しないため除外しています。各除外とその根拠は
[`tests/sigma_lint/validators.yml`](tests/sigma_lint/validators.yml)に文書化されており、上記のリントテストが
強制します。

### Sigma バックエンド移植性

`sigma/correlation/` ルールはアトミックな基本ルールを `id` で参照するため、Sigma 相関変換をサポートする
バックエンドでのみ変換されます。そのサポートはバックエンドごとに異なるため、`-t` の選択が重要です。下の表は固定した
基準(`sigma-cli` 3.1.0、互換性のある最新バックエンド)で測定したもので、
[`tests/sigma_backends/run.sh`](tests/sigma_backends/run.sh)が再現します。

- **ツリー全体(アトミック + 相関)を変換:** Splunk(`-t splunk`)、Elasticsearch EQL(`-t eql`)、
  Grafana Loki(`-t loki`)。`detections/sigma/` をそのまま変換すれば、相関クエリも併せて得られます。
- **アトミックルールのみ(相関は未サポート):** Elasticsearch Lucene(`-t lucene`)、
  OpenSearch(`-t opensearch_lucene`)、そして Sentinel・Defender XDR を対象とする Microsoft `kusto`
  バックエンド(`-t kusto`)。これらでは 5 つのアトミックルールを変換し、相関の時間ウィンドウは製品内でネイティブに
  表現します(例: Sentinel のスケジュール分析の `summarize ... by bin(TimeGenerated, 30m)`)。ディレクトリ全体を
  渡すと "Backend does not support correlation rules" で変換が止まります。

```sh
# アトミックルールのみ、例: Microsoft Sentinel / Defender (kusto バックエンド)
sigma plugin install kusto
sigma convert -t kusto --without-pipeline \
  detections/sigma/artex_enrich_user_agent.yml \
  detections/sigma/artex_selfupdate_egress.yml \
  detections/sigma/artex_guard_audit_framing.yml \
  detections/sigma/artex_recording_proxy_ca.yml \
  detections/sigma/destructive_command_hunting.yml
```

固定したバージョンでの既知の境界: Elasticsearch ES|QL ターゲット(`-t esql`)はガードマーカールールを拒否するため
(`String value expressions are not supported`)、そこでは残りの 3 つのアトミックルールを変換してください。また
IBM QRadar プラグイン(`ibm-qradar-aql`)は固定した pySigma と互換性がなく `--force-install` が
必要なため、テストでは扱いません。環境にインストールされているバックエンドは `sigma list targets` で確認してください。

上記の例は `--without-pipeline` を使い、ルール本文の汎用フィールド名(`cs-user-agent`・`cs-host`・
`CommandLine`)をそのまま出力します。製品スキーマに合わせるには、そのフラグを外して `-p` で処理
パイプラインを適用してください(`sigma list pipelines` を参照)。ただし、製品パイプラインはフィールド名をマッピングしますが、
ルールの汎用 `logsource` が指定しない対象テーブルを追加で要求する場合があります。たとえば
`-p sentinel_asim` は、データに合った `query_table` を設定するまで "Unable to determine table name"
で止まるため、デプロイ前にフィールドと宛先テーブルを環境に合わせてマッピングしてください。

## 貢献

検知とハードニングへの貢献を歓迎します。新しいルールは、すべての指標を観測可能な事実に基づかせ、限界を
`description` に明記し、SigmaHQ バリデーターのベースラインをクリーンに通過し
(`sigma check --validation-config tests/sigma_lint/validators.yml`)、攻撃の手引きと読める内容を
含まないようにしてください。[`../CONTRIBUTING.md`](../CONTRIBUTING.md)を参照してください。
