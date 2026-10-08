# ARTEX 検知ルールテスト

日本語 · [English](README.md)

[`../`](../) 配下の検知ルールが実際に発火するか、そして同じくらい重要な、良性(benign)
トラフィックには沈黙するかを、再現可能な形で証明する回帰テストです。動かして確かめられない検知ルールは
主張にすぎません。これらのテストは、ルールファイルと防御ガイドに書かれた主張を、レビュアーがソースから
再実行して確かめられるものに変えます。

バイナリのパケットキャプチャはリポジトリに含めません。キャプチャは**実行のたびに決定論的に生成**し、終了後に
削除するため、テストは不透明な固定ファイルではなく読めるソースとして配布され、リポジトリを肥大化させ
ません。

## すべてのスイートを一度に実行: [`run-all.sh`](run-all.sh)

[`run-all.sh`](run-all.sh) は、以下の 8 つのスイートを CI と同じ順序で 1 つのコマンドですべて実行するため、8 つの
`run.sh` スクリプトを手作業で 1 つずつ呼び出す必要がありません。前のスイートが失敗しても各スイートは最後まで
実行され、スクリプトは最後にスイートごとの PASS/FAIL 1 行サマリを出力し、1 つでも失敗すれば 0 以外の
コードで終了します。

スイートを実行する前に、ハーネスの自己点検([`check-harness-sync.sh`](check-harness-sync.sh))を先に実行します。
この点検は、上記のスイート一覧、[CI](../../.github/workflows/detections.yml) のスイートごとのステップ、ディスク上のスイート
ディレクトリの 3 つが、異なるスイートや異なる順序を指している場合に、実行を失敗で終わらせます。これは 8 つのスイートが
自分では見えない唯一の空白です。3 か所のうち 1 か所にしか配線されていないスイート(例: `run-all.sh` の項目なしに CI
ステップだけを追加したり、どちらにも入れていないディレクトリ)は、スイートごとのテストをすべて通過しながらも、ローカルで
緑だった `run-all.sh` がもはや緑の CI を意味しなくなります。この点検は 9 番目のスイートではなくゲートであり、
下のサマリには現れないため、検知スイートは 8 つのままです。

```sh
detections/tests/run-all.sh
```

期待される出力(抜粋):

```
===== detection suites summary =====
  PASS  sigma
  PASS  sigma_match
  PASS  sigma_lint
  PASS  sigma_backends
  PASS  suricata
  PASS  attack
  PASS  indicators
  PASS  misp
RESULT: PASS
```

失敗が 1 つでもあれば 0 以外のコードで終了するため、pre-commit フックにそのまま組み込めます。すぐに使える
例がリポジトリ最上位の [`.pre-commit-config.yaml`](../../.pre-commit-config.yaml) にあります。
`pip install pre-commit && pre-commit install` でインストールすると、検知ルールやそのルールが固定したアップストリームソース
ファイルに触れるコミットでランナーが発火します。CI と同じ範囲です。個別スイートが認識するイメージ・バージョンの
オーバーライド(`PYTHON_IMAGE`、`SIGMA_CLI_VERSION`、`SIGMAHQ_VALIDATORS_VERSION`、`SURICATA_IMAGE`)は、ランナーが
そのまま引き継ぐため、どれを export してもすべてのスイートに一括で適用されます。

## Suricata: [`suricata/`](suricata/)

[`suricata/run.sh`](suricata/run.sh) は [`../suricata/artex.rules`](../suricata/artex.rules) の
ネットワークルールをエンドツーエンドで動かし、5 つの性質をアサートします:

- **有効性**: ルールファイル全体が `suricata -T --init-errors-fatal` でロードされるため、以下のどのキャプチャにも
  触れないルールであっても、パース・初期化に失敗すれば検出します。単なる `suricata -r` はそうしたルールを
  スキップしても 0 で終了するため、このロード検査は Sigma スイートの `sigma check` 有効性アサーションに相当する
  Suricata 側の仕組みです。
- **存在性(エンリッチプローバー)**: sid `1000001` がエンリッチプローブごとにちょうど 1 回発火します。
- **速度**: 送信元あたり 300 秒に 30 リクエストという `detection_filter` のしきい値を超えると sid `1000002` が
  発火します。
- **存在性(WebFetch)**: sid `1000003` が norma WebFetch リクエストごとにちょうど 1 回発火し、同じキャプチャで
  エンリッチプローバーの sid は沈黙します。2 つのネットワークシグネチャがそれぞれ発火するだけでなく、互いに特異的で
  あることを確認します。
- **特異性**: 他はすべて同じで User-Agent だけを良性(benign)なブラウザに変えたキャプチャは、ARTEX アラートを
  **1 つも**出しません。

[`suricata/gen_pcap.py`](suricata/gen_pcap.py) は [scapy](https://scapy.net) でキャプチャを作ります。
固定された 1 つの送信元から出る N 個の独立した平文 HTTP リクエスト/レスポンスのフローを、それぞれ指定した User-Agent を
載せて、固定された基準タイムスタンプから 1 秒間隔で配置します。ファイルを書くだけで、パケットを送信したり
ネットワークに触れたりはしません。

### 実行

Docker さえあれば動作します。scapy も Suricata もコンテナで動きます。

```sh
detections/tests/suricata/run.sh
```

期待される出力(抜粋):

```
  PASS  ruleset loads with zero parse/init errors (suricata -T)
  PASS  sid 1000001 presence: one alert per probe  (got 35, want eq 35)
  PASS  sid 1000002 velocity: fires past 30-in-300s  (got 5, want ge 1)
  PASS  sid 1000003 presence: one alert per WebFetch request  (got 8, want eq 8)
  PASS  enrich sids stay silent on norma traffic (specificity)  (got 0, want eq 0)
  PASS  benign browser UA produces no ARTEX alerts  (got 0, want eq 0)
RESULT: PASS
```

アサーションが 1 つでも失敗するとスクリプトは 0 以外のコードで終了するため、CI や pre-commit フックにそのまま組み込めます。
社内にミラーを置いている場合は `SURICATA_IMAGE` / `PYTHON_IMAGE` でイメージを上書きしてください。

### 速度アラート数を正確な値ではなく下限でアサートする理由

`run.sh` は存在性アラート数(`1000001 == 35`・`1000003 == 8`)と良性アラート数(`== 0`)を正確にアサートします。
これらはエンジンのバージョンに依存しないためです。一致するリクエストごとにアラート 1 つ、別の User-Agent には不一致です。速度
ルールのアラート数は、特定の Suricata リリースが境界で `detection_filter` のしきい値をどう扱うかに依存するため、
テストは `>= 1` でアサートし、基準値は別に記録します。**Suricata 8.0.7** では、基準の
実行が sid `1000002` にアラート **5** 件を出します(300 秒に 30 のしきい値を超えた後の 31–35 番目のフロー)。

## Sigma: [`sigma/`](sigma/)

[`sigma/run.sh`](sigma/run.sh) は [`../sigma/`](../sigma/) 配下の Sigma ルールを構造的に、そして
[sigma-cli](https://github.com/SigmaHQ/sigma-cli)(pySigma)でコンパイルして検証し、5 つの性質を
アサートします:

- **有効性**: `sigma check` がツリー全体でエラー 0、条件エラー 0、イシュー 0 を報告します。
- **コンパイル**: `sigma convert -t splunk` がツリー全体をエラーなしでバックエンドのクエリ言語に変換します。
- **指標の保存**: 各アトミック指標の文字列(`artex-enrich/1.0`、`artex-selfupdate`、ガードマーカー、そして記録用
  プロキシの CA ファイル名 `mitmproxy-ca-cert.pem`)がコンパイル済みクエリにそのまま残っているため、ルールが自身が
  基づく文字列を黙って失うことはありません。
- **相関ルールのコンパイル**: [`../sigma/correlation/`](../sigma/correlation/) の行動ルールが捨てられず、
  `event_count` / `value_count` の集計を出力します。
- **相関ルールが実際に機能する**: 相関ルール 1 つだけを*単独で*変換すると失敗します。そのルールがアトミックな
  基本ルールを `id` で参照しているためで、この参照は飾りではなく強制されます。これは上記の Suricata の
  特異性アサーションに相当する Sigma 側の仕組みです。

これは [`../README.ko.md`](../README.ko.md) で説明した構造 + コンパイル検証を、実行可能でアサートする形にした
ものです。後述の対となるスイート [`sigma_match/`](sigma_match/) が、アトミックルールと相関ルールの両方に*マッチング*の半分を
加えます。代表的な悪意のあるイベント(またはタイムライン)が各ルールを発火させ、正常なイベントは発火させないことを
確認するため、Sigma ルールも Suricata ルールと同様に、再現可能な検証テストと再現可能なマッチングテストを
併せ持ちます。(雑な手作りのマッチャーがルールの価値を
損なうという従来の懸念は、パースをすべて pySigma に委譲して解消しました。信頼モデルは次の節で説明します。)

### 実行

Docker さえあれば動作します。sigma-cli と splunk バックエンドがコンテナで動き、リポジトリには何も書き込み
ません。

```sh
detections/tests/sigma/run.sh
```

期待される出力(抜粋):

```
  PASS  sigma check: 0 errors, 0 condition errors, 0 issues
  PASS  whole tree converts to splunk (exit 0)
  PASS  indicator present: artex-enrich/1.0
  PASS  correlation rule fails to convert alone — it requires its atomic base rule
RESULT: PASS
```

アサーションが 1 つでも失敗するとスクリプトは 0 以外のコードで終了するため、CI や pre-commit フックにそのまま組み込めます。
sigma-cli は基準バージョン(`3.1.0`)に固定されています。社内にミラーを置いている場合は
`SIGMA_CLI_VERSION` でバージョンを、`PYTHON_IMAGE` でイメージを上書きしてください。

## Sigma リアルタイムイベントマッチング: [`sigma_match/`](sigma_match/)

[`sigma_match/run.sh`](sigma_match/run.sh) は、[`../sigma/`](../sigma/) 配下の Sigma ルールが、アトミックルールと
[`../sigma/correlation/`](../sigma/correlation/) の相関ルールの両方を含めて、一致するイベントに実際に*発火*し、
正常なイベントには沈黙することを証明します。Suricata スイートがネットワークルールに与える「動かして確かめられない検知ルールは
主張にすぎない」という保証を、ホスト・ログ層のルールに拡張したものです。アトミックルールのセットと相関ルールのセットについて、合わせて 6 つの
性質をアサートします:

- **ルール・サンプルの対応付け**: すべてのアトミックルールには [`events/<名前>.json`](sigma_match/events/) のサンプルファイルがあり、
  すべてのサンプルファイルはルールに辿れます。サンプルなしで追加したルールは、検証を受けずに通過するのではなく、ここで
  失敗します。
- **真陽性(true positive)**: 各ルールが自身の悪意のあるサンプルイベントをすべてマッチします。
- **真陰性(true negative)**: 各ルールが自身の正常なサンプルイベントを 1 つもマッチしません。たとえば
  `.mitmproxy/` 配下の単独の `mitmproxy-ca-cert.pem` は、記録用プロキシルールを発火させ**ません**。その
  ルールの `|all` 修飾子が、ARTEX が使う `_ca/` ディレクトリまで併せて要求するためで、この判別を証明するのが
  まさにマッチングテストです。
- **相関ルール・タイムラインの対応付け**: すべての相関ルールには [`events/correlation/<名前>.json`](sigma_match/events/correlation/)
  のタイムラインファイルがあり、すべてのタイムラインはルールに辿れます。タイムラインの各イベントは相対秒を含む `ts`
  フィールドを持ちます。
- **相関ルールの真陽性**: しきい値を時間ウィンドウ内に 1 グループが満たす陽性タイムラインで各ルールが発火します。
  たとえば 1 つの送信元(`c-ip`)から 10 分以内に異なる 20 のホストへ広がるリクエストが、エンリッチファンアウトルールを
  発火させます。
- **相関ルールの真陰性**: しきい値未満、しきい値は満たしたが時間ウィンドウを外れた場合、グループが分かれた場合、
  時間相関で片方のレッグが欠けた場合は沈黙します。特にリクエスト量は多くても幅(異なるホスト数)が小さい
  バーストは、ファンアウトルールを発火させ**ません**。幅がシグナルであり量ではなく、この判別を証明するのが
  まさにマッチングテストです。

信頼モデルは次のとおりです。手書きのコードではなく pySigma が各ルールをパースします。アトミックルールは修飾子と
条件をツリーにコンパイルし(`|contains` → ワイルドカード値、`|all` → AND、`1 of selection_*` → OR)、相関ルールは
集計仕様(種類・group-by・時間ウィンドウ・しきい値条件・参照するアトミックルール)にコンパイルします。[`check.py`](sigma_match/check.py)
はそのツリーと仕様を辿るだけで、相関ルールがどのイベントを取り込むかはアトミックルールとまったく同じマッチャーで
判定するため、権威ある Sigma ロジックは pySigma の中に残ります。明示的にサポートしない構文に出会うと、黙って
通過させず例外を投げます(fail-closed)。範囲と限界はスクリプトの冒頭に明記してあります。相関ルールの
時間ウィンドウは標準的なスライディングウィンドウ(マッチしたイベントごとに `timespan` の長さのウィンドウを取る)の解釈であり、実際の SIEM のウィンドウ
方式は異なる場合があります。マッチングは**大文字小文字を区別せず**(`sigma/` スイートが対象とする splunk バックエンドの
既定値であり、破壊コマンドルールの誤検知コメント自体がこれを前提としています)、キーワードマッチングは全文の部分文字列検索です。
これはルールのフィールド・値・条件・集計ロジックに対する回帰テストであり、フィールド正規化が異なりうるそれぞれの SIEM で
検証する作業の代わりにはなりません。

### 実行

Docker さえあれば動作します。pySigma がコンテナで動き、リポジトリには何も書き込みません。

```sh
detections/tests/sigma_match/run.sh
```

期待される出力(抜粋):

```
  PASS  rule/sample pairing: 5 atomic rules, 5 event files, no orphans
  PASS  artex_enrich_user_agent: 1/1 positive events matched
  PASS  artex_recording_proxy_ca: 2/2 benign events correctly not matched
  PASS  rule/timeline pairing: 4 correlation rules, 4 timeline files, no orphans
  PASS  artex_enrich_fanout: fired — 20 distinct hosts from one source within the 10-minute window
  PASS  artex_enrich_fanout: quiet — high volume, low breadth: 25 requests from one source but only 4 distinct hosts
RESULT: PASS
```

アサーションが 1 つでも失敗するとスクリプトは 0 以外のコードで終了するため、CI や pre-commit フックにそのまま組み込めます。
pySigma は基準バージョン(`2.0.0`)に固定されています。社内にミラーを置いている場合は `PYSIGMA_VERSION`
でバージョンを、`PYTHON_IMAGE` でイメージを上書きしてください。

## Sigma バックエンド移植性: [`sigma_backends/`](sigma_backends/)

[`sigma_backends/run.sh`](sigma_backends/run.sh) は、ルールが Sigma テストが動かす単一の Splunk
の例を超えて変換されることを証明し、[`../README.ko.md`](../README.ko.md) のバックエンド別サポート表を正直に
保ちます。Sigma 相関ルールの変換はバックエンドによって異なるため、README はどの `-t` ターゲットがツリー全体を
受け付け、どのターゲットがアトミックルールのみを受け付けるかを防御者に伝えます。再実行して初めて信じられる
主張です。2 つの性質をアサートしますが、どちらも肯定形なので、実際の回帰があるときだけ失敗します:

- **相関ルールの移植性**: ツリー全体(アトミック + 相関)が Splunk、Elasticsearch `eql` ターゲット、Grafana
  `loki` で変換され、エンリッチ指標が各クエリにそのまま残ります。相関ルールが Splunk 専用ではないことを
  示します。
- **アトミックのみのフォールバック動作**: 5 つのアトミックルールは `lucene` と Microsoft `kusto` バックエンドでも
  変換されます。これらのバックエンドは固定されたバージョンで Sigma 相関ルールの変換をサポートしないため、該当バックエンドを
  使う防御者はアトミックルールをデプロイし、相関ウィンドウをそのバックエンド固有の機能で表現できます。

「バックエンド X は相関ルールを処理できない」という否定形は、あえてアサートしません。そうするとバックエンドが
*改善される*ことが赤いビルドになってしまうためです。正直な限界は README に記載してあり、このテストのコマンドが
それを再現します。[`sigma_backends/check.sh`](sigma_backends/check.sh) はコンテナ内側の半分です。
固定された sigma-cli と 4 つのバックエンドをインストールし、読み取り専用でマウントしたルールツリーを読みます。

### 実行

Docker さえあれば動作します。sigma-cli とバックエンドがコンテナで動き、リポジトリには何も書き込み
ません。

```sh
detections/tests/sigma_backends/run.sh
```

期待される出力(抜粋):

```
  PASS  whole tree (atomic + correlation) converts on 'eql', enrich indicator survives
  PASS  five atomic rules convert on 'kusto', enrich indicator survives
RESULT: PASS
```

アサーションが 1 つでも失敗するとスクリプトは 0 以外のコードで終了します。sigma-cli は固定されており(`3.1.0`、
`SIGMA_CLI_VERSION` で上書き可能)、バックエンドプラグインは互換性のある最新バージョンでインストールされます。そのため、この
スイートはアップストリームのバックエンドリリースに最も敏感です。サポートを落としたプラグインはビルドを赤くし、これは
固定バージョンと README の表を併せて更新せよというシグナルです。

## SigmaHQ 慣例リント: [`sigma_lint/`](sigma_lint/)

[`sigma_lint/run.sh`](sigma_lint/run.sh) は、README と `CONTRIBUTING.md` の「`sigma check` をクリーンに
通過する」という約束が、pySigma のコア検査だけでなく SigmaHQ の慣例まで含むようにします。単なる
`sigma check` は `pySigma-validators-sigmahq` プラグインをロードしないため、タイトルの大文字小文字・フィールド名の分類
体系・ログソースの分類体系・参照リンクの慣例が検査されずに通過します。このスイートはそのプラグインをインストールし、
[`sigma_lint/validators.yml`](sigma_lint/validators.yml) に文書化したベースラインに合わせて、全検査セットを
実行します。2 つの性質をアサートします:

- **文書化したベースラインがクリーン**: `validators.yml` を付けて `sigma check` を実行すると、エラー 0、イシュー 0 を
  報告します。
- **全セットが生きており、文書化した除外だけが残る**: 除外なしで SigmaHQ のすべてのバリデーターを実行してもイシューが
  報告され、その各々は `validators.yml` が意図的にオフにする 4 つの検査のいずれかです(それ以外はありません)。これは
  空虚さを防ぐガードです。プラグインのロードに失敗していた場合、全実行は何も報告せず、1 つ目の性質が
  誤った理由で通過してしまうため、既知の除外項目が必ず現れることを要求します。

4 つの除外項目は、SigmaHQ のモノレポのファイル整理方式(ログソースのプレフィックスが付いたファイル名と `correlation_`
ファイル名)と分類体系(汎用の `application` ログソース、製品名のない `process_creation`)、そしてブランチ対
パーマリンク(permalink)の参照慣例を含みます。どれも、自身のリポジトリの生きたドキュメントを参照する小さく
独立したルールセットには合いません。各除外項目には、その根拠を `validators.yml` の中に併せて記載して
あります。*残り*のすべての SigmaHQ 検査は強制されるため、新たな慣例違反を持ち込んだルール(大文字小文字が誤ったタイトル、
分類体系を外れたフィールド名)はビルドを赤くします。`pySigma-validators-sigmahq` は固定されており
(`0.21.0`、`SIGMAHQ_VALIDATORS_VERSION` で上書き可能)、バージョンを上げると新たな慣例が現れることがありますが、これは
ルールや文書化したベースラインを更新せよというシグナルです。

### 実行

Docker さえあれば動作します。sigma-cli とバリデータープラグインがコンテナで動き、リポジトリには何も書き込み
ません。

```sh
detections/tests/sigma_lint/run.sh
```

期待される出力(抜粋):

```
  PASS  sigma check with the documented baseline: 0 errors, 0 issues
  PASS  every reported issue is one of the four documented exclusions
RESULT: PASS
```

## ATT&CK レイヤー: [`attack/`](attack/)

[`attack/run.sh`](attack/run.sh) は、[`../attack/artex_navigator_layer.json`](../attack/artex_navigator_layer.json)
の [ATT&CK カバレッジレイヤー](../attack/)が、カバーすると主張するルールと食い違っていないかを確認します。ルール
セットとずれたカバレッジレイヤーはないほうがましなので、このテストは「これらのルールがこれらの ATT&CK 技法を
カバーする」を、レビュアーがソースから再実行して確かめられるものに変えます。アサートする内容:

- **有効なレイヤー**: ファイルが JSON としてパースされ、必須の ATT&CK Navigator v4.x フィールドを持ち、すべての
  項目に正しい形式の技法 ID と有効な ATT&CK 戦術があります。
- **双方向の一致**: スコアが付いた技法が Sigma ルールの `attack.*` 技法タグと*正確に*一致します。
  レイヤーに欠けているルール技法も、ルールにないレイヤー技法もありません。戦術も同じ方式で一致します。
- **根拠あり**: スコアが付いたすべての技法の注釈が実在するルールファイルを指すため、レイヤーが名前を
  変更された、または削除されたルールを引用することはできません。

これは発火テストではなく整合性検査です。検知バックエンドが不要で Python 標準ライブラリだけで済むため、Sigma・Suricata
テストと違い、バージョンに依存するアラート数がありません。[`attack/check.py`](attack/check.py)
はコンテナ内側の半分です。読み取り専用でマウントした検知ツリーを読み、何も書き込みません。

### 実行

Docker さえあれば動作します。検査は Python コンテナで動き、リポジトリには何も書き込みません。

```sh
detections/tests/attack/run.sh
```

期待される出力(抜粋):

```
  PASS  scored techniques match the rule set exactly (8: T1059, T1105, T1485, T1489, T1557, T1561.002, T1592, T1595)
  PASS  scored tactics match the rule set exactly (collection, command-and-control, credential-access, execution, impact, reconnaissance)
RESULT: PASS
```

アサーションが 1 つでも失敗するとスクリプトは 0 以外のコードで終了するため、CI や pre-commit フックにそのまま組み込めます。
社内にミラーを置いている場合は `PYTHON_IMAGE` でイメージを上書きしてください。

## 指標の根拠(source-of-truth): [`indicators/`](indicators/)

[`indicators/run.sh`](indicators/run.sh) は、上記 3 つのテストができない 1 つのことを証明します。各ルールが
固定した指標が、今も ARTEX 自身のソースが実際に発する文字列であるかどうかです。Sigma テストは、指標が
ルール→クエリの*コンパイル*を経て残ることを証明し、ATT&CK テストはレイヤーがルールタグと一致することを
証明し、Suricata テストはネットワークルールが生成したキャプチャで*発火*することを証明します。どれも、指標が
由来すると主張するソースファイルを辿り直してはいません。これらがすべて見逃す腐敗は、プローバーの User-Agent
を `artex-enrich/2.0` に上げたり、ガードマーカーを書き直したりするアップストリームの再同期です。それでもルールはすべて
コンパイルされ、レイヤーは依然として一致し、pcap テストも依然として発火します。ところがデプロイされたルールは実際の
ARTEX トラフィックに黙ってマッチしなくなります。各指標について双方向でアサートします:

- **ソースが今も出力している**: 値がそれを生み出すアップストリームのソースファイルに存在します(`enrich/enrich.go`
  の `artex-enrich/1.0`、`selfupdate/` の `artex-selfupdate`、`guard/guard.go` のガードマーカー)。値が
  ないということは、ルールがまだ追いついていないアップストリームの変更を意味します。
- **ルールが今も固定している**: 値がそれを基に構築したルールに存在するため、ルールの編集で指標がソースから
  黙って切り離されることはありません。Suricata ルールは `startswith` プレフィックスで確認しますが、これはそのルールが
  実際にワイヤをマッチさせる方式と同じです。
- **ブロックリストの対応**: 破壊的コマンドのトークン(`rm -rf`、`mkfs`、`DROP DATABASE`、`FLUSHALL`)が ARTEX
  ガードのブロックリスト(`db/db.go`)と、それを反映したハンティングルールの両方に現れます。これらは固有のフィンガープリント
  ではなく汎用的なハンティングの手がかりなので、テストはルールが実際に主張する対応関係だけをアサートします。
- **公開リストが根拠を維持している**: 防御者が持ち帰って使う成果物である機械可読な指標リスト
  [`detections/indicators/artex_indicators.csv`](../indicators/artex_indicators.csv) を行単位で再度
  読みます。すべての値は引用したソースファイルに今も存在し、引用したルールに固定されていなければならず、テストが
  根拠を確認したすべてのフィンガープリントはこのリストに現れなければなりません。そのため公開された CSV は、自身が由来すると
  主張するソースから、どちらの方向にも黙ってずれることはありません。
- **2 つのゲートが固定された各ソースで発火する**: テストが読むすべてのアップストリームソースは、それを動かす 2 つのゲートに
  含まれます。CI ワークフローの `push`・`pull_request` の paths フィルター([`.github/workflows/detections.yml`](../../.github/workflows/detections.yml))と
  ローカル pre-commit フックの `files` 正規表現([`.pre-commit-config.yaml`](../../.pre-commit-config.yaml))です。
  必要な集合は指標自体から導出されるため、新しいソースを固定しながら(以前の `cmd/artex/main.go` のポートが
  そうだったように)*両方*のゲートに配線しないと、ここで失敗します。そうしないと、そのソースだけに触れた変更が
  そのソースを欠いたゲートでテストをスキップします。CI ではマージゲートを緑で通過し、フックでは
  「CI と同じソース範囲」と約束しておきながらローカルで最後まで捕捉されません。

これは [`../README.ko.md`](../README.ko.md) の約束(「ここにあるすべての指標は、推測ではなくこのリポジトリのソースで確認した
文字列に基づく」)と CONTRIBUTING の最初の貢献契約を、レビュアーが再実行できるガードに
変えます。ATT&CK テストと同様に検知バックエンドが不要で Python 標準ライブラリだけで済みます。
[`indicators/check.py`](indicators/check.py) はルールツリー、公開指標リスト、固定されたソースパッケージ、そして
それを発火させる 2 つのゲート(CI ワークフローと pre-commit 設定)を読み取り専用でマウントして読み、何も書き込み
ません。

### 実行

Docker さえあれば動作します。検査は Python コンテナで動き、リポジトリには何も書き込みません。

```sh
detections/tests/indicators/run.sh
```

期待される出力(抜粋):

```
  PASS  enrichment prober User-Agent: 'artex-enrich/1.0' emitted by enrich/enrich.go
  PASS  detections/sigma/artex_enrich_user_agent.yml pins 'artex-enrich/1.0'
  PASS  'FLUSHALL' present in both db/db.go and detections/sigma/destructive_command_hunting.yml
  PASS  enrich-user-agent: 'artex-enrich/1.0' grounded in enrich/enrich.go
  PASS  tested fingerprint 'artex-enrich/1.0' is published in the list
  PASS  .github/workflows/detections.yml push paths covers cmd/artex/main.go
  PASS  .pre-commit-config.yaml files covers cmd/artex/main.go
RESULT: PASS
```

アサーションが 1 つでも失敗するとスクリプトは 0 以外のコードで終了するため、CI や pre-commit フックにそのまま組み込めます。
社内にミラーを置いている場合は `PYTHON_IMAGE` でイメージを上書きしてください。

## MISP エクスポート整合性: [`misp/`](misp/)

[`misp/run.sh`](misp/run.sh) は、指標の 2 つ目の公開形態である、そのままインポートできる MISP イベント
[`detections/indicators/artex_indicators.misp.json`](../indicators/artex_indicators.misp.json) を
扱います。上記の指標テストが CSV をソースに基づかせ続けるのに対し、このテストは、防御者が実際に脅威
インテリジェンスプラットフォームに読み込む成果物である MISP イベントが、その CSV からずれないように保ちます。アサートする内容:

- **本当に MISP である**: イベントが [pymisp](https://github.com/MISP/PyMISP) でロードされますが、そのオブジェクト
  モデルは `type` が本物の MISP タイプではない属性を拒否します。もっともらしく見えても無効なタイプは
  ここで失敗するため、「有効な MISP」は単に主張するのではなく、MISP サーバーが使うライブラリで
  証明されます。
- **CSV と行単位で同期**: すべての CSV 行が、意図したタイプ・カテゴリを持つ MISP 属性ちょうど 1 つに
  対応し(`http.user-agent` → `user-agent`、ガードマーカー `string` → `pattern-in-file`、`port` →
  `port`、`ip-dst|port` → 合成 `ip|port` 値を持つ `ip-dst|port`、探索スキーマ `other` → `other`)、CSV の行なしに残る MISP 属性が
  1 つもありません。イベントは CSV と一緒に手作業で維持するため、`artex_indicators.misp.json` を同じ
  コミットで合わせて更新しないまま CSV の行を追加・削除・タイプ変更すると失敗します。
- **`to_ids` が `rule` 列を反映する**: ルールの裏付けがある指標は `to_ids: true` で、ルールのない
  ホストフォレンジック行は `disable_correlation: true` とともに `to_ids: false` です。CSV が含意する
  ものと異なるようにフラグを反転すると失敗するため、MISP イベントはどのフィンガープリントが実行可能かを黙って
  水増ししたり削ったりして主張することはできません。
- **ガードマーカーがバイト単位で保存され**、`detections/**` が CI の paths フィルターに入っているため、CSV やイベントを
  変更するとこのスイートが発火します。

上記の純粋な標準ライブラリのテストと異なり、このスイートはコンテナ内に固定された `pymisp` を
インストールします(ホストには何もインストールしません)。[`misp/check.py`](misp/check.py) は CSV、MISP イベント、
CI ワークフローを読み取り専用でマウントして読み、何も書き込みません。

### 実行

Docker さえあれば動作します。pymisp がコンテナにインストールされ、リポジトリには何も書き込みません。

```sh
detections/tests/misp/run.sh
```

アサーションが 1 つでも失敗するとスクリプトは 0 以外のコードで終了します。社内にミラーを置いている場合は
`PYTHON_IMAGE` でイメージを、`PYMISP_VERSION` で固定されたライブラリを上書きしてください。

## 貢献

新しい検知ルールは、それが発火することを示すテストがあるとより強力です。テストは自身の入力を決定論的に
生成し、エンジンのバージョンに依存しない性質は正確にアサートし(それより緩い性質は基準値を記録した下限で)、攻撃の
手引きと読まれうる内容は避けてください。[`../../CONTRIBUTING.md`](../../CONTRIBUTING.md) と
[`../README.ko.md`](../README.ko.md) のルール索引を参照してください。

8 つのスイートはすべて、`detections/` に触れるあらゆる push や pull request で CI で動きます
([`../../.github/workflows/detections.yml`](../../.github/workflows/detections.yml) を参照)。そして指標
テストは、それが固定したアップストリームソースファイル(`enrich/`、`selfupdate/`、`guard/`、`db/`、`cmd/artex/main.go`)が
変更されたときも動きます。そのため、指標を落とす、ATT&CK レイヤーからずれる、文書化したバックエンドで
変換が止まる、SigmaHQ の慣例を破る、ソースとの同期がずれる、MISP イベントが CSV からずれたままにする、
あるいはワークフローがまだ監視していない新しいソースを固定する、といったルール変更は、マージされる前にビルドを赤く
します。
