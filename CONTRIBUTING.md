# 貢献ガイド (Contributing)

日本語 · [English](CONTRIBUTING.en.md)

ARTEX 韓国語版(`artex-ko`)に関心をお寄せいただき、ありがとうございます。この文書は、貢献を始める前に
知っておくべき範囲・方針・手順を日本語でまとめたものです。貢献を送る前に、必ず
[使用範囲と法的責任](#使用範囲と法的責任)と[ローカライズ方針](#ローカライズ方針)を先にお読みください。

- バグを報告したり機能を提案したりするには → [Issue テンプレート](https://github.com/jiwoochris/artex-ko/issues/new/choose)を使用してください。
- 翻訳・ローカライズの誤りを見つけた場合 → 「翻訳・ローカライズエラー」の Issue テンプレートを使用してください。
- セキュリティ脆弱性を見つけた場合 → **公開 Issue として投稿せず**、[SECURITY.md](SECURITY.md)の手順に従ってください。
- すべての参加者は[行動規範(CODE_OF_CONDUCT.md)](CODE_OF_CONDUCT.md)を守る必要があります。

---

## 使用範囲と法的責任

ARTEX は、LLM マルチエージェントが**自律的に**ペネトレーションテストを実行する攻撃的セキュリティツールです。
貢献者も、ユーザーとまったく同じ範囲の制限を受けます。

- コードを検証する際は、**自分が所有している、または書面で明示的な許可を得た対象**、あるいは
  **ローカルの隔離環境**(例: Docker で起動した OWASP Juice Shop や DVWA のような、意図的に脆弱で
  自分が所有する対象)に対してのみツールを実行してください。
- 許可された範囲を超えて、実際の本番・リモートシステムに対しスキャン・検出・エクスプロイトを行うコード、
  またはそのような使用を助長する変更は受け付けません。
- 大韓民国において、権限なく他人の情報通信網に侵入したり障害を引き起こしたりする行為は
  「情報通信網利用促進及び情報保護等に関する法律」違反であり、収集・露出される個人情報は
  「個人情報保護法」の適用を受けます。詳しい告知は [README](README.md#️-最初にお読みください--利用範囲と国内法に関する告知)にあります。

貢献として提出したコード・文書がどのように使われるかについての法的責任は、それを実行するユーザー本人が
負います。このリポジトリは「現状有姿(AS IS)」で提供されます。

---

## ローカライズ方針

このリポジトリの存在理由は、原本の [Autumn-27/ARTEX](https://github.com/Autumn-27/ARTEX) の
**判断性能をそのまま保ちながら、ユーザーに見える成果物だけを韓国語にすること**です。
この方針から外れる翻訳の貢献は、性能を低下させるおそれがあるため受け付けません。

- **エージェントの内部推論プロンプト(行動指針の本文)は翻訳しないでください。** 原文(中国語)で
  ベンチマークされた動作を維持する必要があります。この本文は `agent/promptcatalog.go` と DB シード
  (`agent_prompts`)にあります。翻訳はエージェントの判断にドリフトを引き起こします。
- **ユーザーに公開される成果物のみを韓国語に強制します。** 検出結果(`report_finding`)、
  事実の要約(`record_fact`)、最終レポート、対話の応答が該当します。この強制は
  `agent/prompt.go` の `langDirective()` というコード固定の末尾として、各ロールの system
  プロンプトの末尾に付加されます。出力言語を変えるには、この関数を修正してください。
- **コマンド・ペイロード・コード・URL・ログの原文は翻訳しません。** 分析に必要な原本ですので、
  そのまま残します。
- **原本の中国語は保存します。** 文書は `README.zh.md`、UI 文字列は `web/messages/zh.json`
  に原文をそのまま残し、上流(upstream)リポジトリの変更と照合しやすくします。韓国語
  訳は `web/messages/ko.json` に入れます。
- UI 文字列を新たに翻訳するときは、ハードコーディングせず、メッセージファイルのキーとして追加してください。
- **出力言語の強制はハードキャップではなく、プロンプトによる誘導です。** `langDirective()` は出力言語を
  韓国語にするよう**指示**するだけで、強制的に固定するわけではありません。そのため、韓国語の忠実度はモデルの能力・ロール・
  文脈によって変わります。ローカライズの変更を検証するときは、能力のあるモデル(フロンティア級)を使ってください。
  低価格・小型モデルは、レポートや要約が原文(中国語)に戻ってしまうことがあるため、翻訳が正しく適用されたかを
  低価格モデルの出力だけで判断しないでください。OpenAI 系モデルを使う際の `max_tokens` 設定の落とし穴は
  [README の「モデルの選択と出力言語」節](README.md#モデル選択と出力言語)にまとめてあります。
- **上流(upstream)の変更に追従する手順は、メンテナー向けの案内文書にあります。** 原本の ARTEX が
  更新されたときに、保存対象の資産と翻訳対象を仕分けて反映し、翻訳の対称性とドリフトを検査するランブックは
  [MAINTAINING.md](MAINTAINING.md)にまとめられています。

---

## 開発環境

このプロジェクトは、**Go バックエンド**(単一バイナリにフロントエンドを内蔵)+ **Next.js フロントエンド**で
構成されます。

主要機能の設計意図は `docs/` の設計文書にまとめられています。脆弱性とトラフィックの証拠を
結び付ける機能(レポートエージェントの自動バインディング、`report_finding` の `traffic_refs` など)を
扱う場合は、まず[脆弱性の複数トラフィック証拠の設計文書](docs/finding-traffic-evidence-ko.md)を
お読みください。原文(中国語)は同じフォルダの `finding-traffic-evidence-zh.md` に保存されています。

### 必要なバージョン

- Go 1.26 以上(`go.mod` 基準)
- Node.js 22 以上(リリースワークフロー基準)
- Docker と Docker Compose(ローカルでの実行・検証用)

### バックエンド (Go)

ローカルに Go がインストールされている場合は、リポジトリのルートで次を実行します。

```bash
go build ./...
go vet ./agent/
go test ./agent/
```

ローカルに Go がない場合は、Docker で同様に検証できます。モジュール・ビルドキャッシュを named volume
に置くと、再実行が速くなります。

```bash
docker run --rm -v "$PWD":/src -w /src \
  -v artexko-gomod:/go/pkg/mod -v artexko-gocache:/root/.cache/go-build \
  golang:1.26 sh -c 'go build ./... && go vet ./agent/ && go test ./agent/'
```

#### DB 統合テスト (postgres が必要)

上記の `go test ./agent/` は、PostgreSQL に接続しないと動かない **DB 統合テストを黙ってスキップします。**
`agent`・`config`・`db`・`evidence`・`llmrec`・`server` の六つのパッケージには、実際の
データベースが必要なテストが含まれていますが、環境変数 `ARTEX_PG_DSN` がなく、設定
ファイルにも `database` 項目がなければ、それらのテストは `--- SKIP` で通過し、パッケージは `ok` で
終わります。そのため、この六つのパッケージを修正した後に DSN なしで検証すると、**ローカルは通過(ok)するのに PR の
`go-db` ジョブは失敗する**ことがあります。

これらのテストをローカルで動かすには、PostgreSQL を起動して `ARTEX_PG_DSN` を渡します。以下は
CI と同じ `postgres:16-alpine` を隔離ネットワークで起動して実行する例で、上記と同じ named
volume を再利用します。

```bash
# 1) 隔離ネットワークと空の postgres を起動します (CI と同じイメージ・アカウント)。
docker network create artexko-db 2>/dev/null || true
docker run -d --name artexko-pg --network artexko-db \
  -e POSTGRES_USER=artex -e POSTGRES_PASSWORD=artex -e POSTGRES_DB=artex \
  postgres:16-alpine
until docker exec artexko-pg pg_isready -U artex -d artex >/dev/null 2>&1; do sleep 1; done

# 2) DSN を渡して DB 統合パッケージを実行します (DSN の host はコンテナ名です)。
#    修正したパッケージだけを実行するには、./agent/ の部分を config・db・evidence・llmrec・server に置き換えます。
docker run --rm --network artexko-db -v "$PWD":/src -w /src \
  -v artexko-gomod:/go/pkg/mod -v artexko-gocache:/root/.cache/go-build \
  -e ARTEX_PG_DSN='postgres://artex:artex@artexko-pg:5432/artex?sslmode=disable' \
  golang:1.26 sh -c 'go test ./agent/ -count=1'

# 3) 後片付けをします。
docker rm -f artexko-pg && docker network rm artexko-db
```

CI の `go-db` ジョブは、この六つのパッケージを**それぞれ専用の postgres で隔離して**、マージ前に強制的に
実行します(`.github/workflows/ci.yml`)。DB 統合パッケージを修正した場合は、PR を出す前に上記の
方法で該当パッケージを直接確認することをお勧めします。

### フロントエンド (web)

```bash
cd web
npm ci
npm run dev          # 開発サーバー
npm run build        # プロダクションビルド
npm run build:static # 静的エクスポートビルド(マージゲート · TypeScript の型チェックを含む)
npm run check        # Biome のリント・フォーマット検査(情報用 · 既存の負債のためまだマージゲートではない)
npm run check:fix    # 自動修正
```

コミット前のフォーマット・リントは Biome で管理します。`lint-staged` がステージングされたファイルに対して
`biome check --write` を自動で実行します。

### 全体の実行 (Docker Compose)

```bash
cp .env.example .env     # POSTGRES_PASSWORD を設定
docker compose up -d     # artex + postgres を起動 → http://localhost:8787
```

---

## 貢献手順

1. まず **Issue を立てます。** 大きな変更は、作業を始める前に Issue で方向性を
   合わせておくのがよいでしょう。小さな修正(誤字・リンク・明らかなバグ)は、すぐに PR を送っても構いません。
2. リポジトリを**フォーク**してトピックブランチを作ります。ブランチ名は `feat/...`、`fix/...`、
   `docs/...`、`i18n/...` のように、変更の性格を先頭に置きます。
3. 変更を書き、**該当範囲の検証を自分で実行します。** Go の変更であれば、上記の
   `build`・`vet`・`test` を通します。web の変更であれば `npm run build:static`
   (マージゲート · TypeScript の型チェックも併せて行います)を通します。
   `npm run check`(Biome)は、上流から引き継いだ既存のリント負債が残っているため、まだマージ
   ゲートではなく、`web.yml` では情報用のステップとしてのみ実行されるので、
   全体を通す必要はありません。代わりに、**自分の変更が新たなエラーを加えていないか**だけを確認すれば十分です(コミット時に
   `lint-staged` がステージングしたファイルにのみ `biome check --write` を自動適用します)。
   文書(`.md`)を変更した場合は、`python3 -I scripts/check-doc-links.py` で、リポジトリ内の
   リンク・画像参照と文書アンカー(`#見出し`)リンクが壊れていないかを確認します。アンカーは
   GitHub と同じ規則で見出しから slug を作って照合するため、見出しの文字を変えながら
   その見出しを指していたアンカーリンクを一緒に直さないと、ここで引っかかります(CI の `docs`
   ワークフローが同じ検査をマージゲートとして強制します)。この検査はリポジトリルートの
   [`.pre-commit-config.yaml`](.pre-commit-config.yaml)にも `docs` フックとして入っており、
   `pre-commit install` をしておけばコミット時に自動で実行されます(Python 標準ライブラリだけを
   使い、ネットワークに接続しないので、Docker なしで完了します)。
4. **PR を開きます。** タイトル・説明は [PR テンプレート](.github/PULL_REQUEST_TEMPLATE.md)に従い、
   何をなぜ変えたのかと、どのように検証したのかを書きます。UI を変更した場合はスクリーンショットを添付します。
5. ユーザーに見える変更(機能・ローカライズ・文書・検出ルールなど)であれば、[変更履歴(CHANGELOG.md)](CHANGELOG.md)
   の `[Unreleased]` 節に一行を加えます。内部リファクタリングやテスト専用の変更は省略しても構いません。

### コミットメッセージ

既存のコミット履歴の慣例に従います。形式は `type(scope): 説明` で、説明は韓国語で書きます。

- `type`: `feat` · `fix` · `docs` · `chore` · `refactor` · `test` · `i18n` など
- `scope`: 変更された領域(`agent` · `web` · `server` など)、省略可

例です。

```
feat(agent): 사용자 노출 출력을 한국어로 강제 (langDirective)
docs: 한국어 README 작성, 원본은 README.zh.md 로 보존
i18n(web): 대시보드 네비게이션 라벨 한국어 번역
```

---

## 検出ルール・検出テストへの貢献

このリポジトリは、ARTEX のような自律型 AI 攻撃を**防御・検出**するためのルールを [`detections/`](detections/)にあわせて
置いています。配布可能な [Sigma](https://sigmahq.io) ルール([`detections/sigma/`](detections/sigma/))、ネットワーク向けの
[Suricata](https://suricata.io) ルール([`detections/suricata/`](detections/suricata/))、
[MITRE ATT&CK](https://attack.mitre.org/) カバレッジレイヤー([`detections/attack/`](detections/attack/))、そして
これらのルールが実際に発火することを再現可能な形で証明するテスト([`detections/tests/`](detections/tests/))で
構成されています。検出ルールを新しく送る、または修正する際は、以下の契約を守ってください。八つのテストスイートがこの
契約の大部分を機械的に強制するため、ルールだけを変更してテスト・レイヤーを更新しないと、テストが失敗します。

- **すべての指標を観測可能な事実に接地します。** ルールが使う文字列・User-Agent・行動のしきい値は、このリポジトリの
  ソースで実際に確認できるものでなければならず、推測で作ってはいけません。根拠となるソースファイルをルールの中に
  明記してください(例: `artex-enrich/1.0` 指標は `enrich/enrich.go` で確認できます)。指標一致テスト
  ([`detections/tests/indicators/`](detections/tests/indicators/))が、各指標が上流ソースとルールの両方に
  今も存在するかを検査するため、上流の再同期でソース文字列が変わった場合、ルールも一緒に修正しない限りテストが失敗します。
  機械可読な指標一覧([`detections/indicators/artex_indicators.csv`](detections/indicators/artex_indicators.csv))を
  変更したら、その指標をそのまま収めた MISP イベント([`detections/indicators/artex_indicators.misp.json`](detections/indicators/artex_indicators.misp.json))も
  あわせて更新します。MISP エクスポートテスト([`detections/tests/misp/`](detections/tests/misp/))が、二つのファイルが行
  単位で一致するか、そしてそのイベントが pymisp で読み込める有効な MISP ドキュメントであるかを強制します。
- **限界を正直に書きます。** Sigma ルールは `description` に、Suricata ルールはコメントに、そのルールが捕捉できない
  ケースと誤検知の可能性を書きます。ARTEX 固有のシグネチャではなく一般的なハンティングの手がかり(例: 破壊的コマンド)である場合は、そう
  明記し、一度のヒットだけで攻撃者を ARTEX と断定しないようにします。
- **静的検証を通します。** Sigma ルールは、SigmaHQ バリデーター基準で問題 0 件で通過しなければなりません
  (`sigma check --validation-config detections/tests/sigma_lint/validators.yml`)。既定の `sigma check` は
  pySigma のコア検証器のみを実行するため、タイトル表記・フィールド/ログソース分類・参照リンクのような SigmaHQ の慣例は、この基準でのみ
  フィルタリングされます。四つの例外は、単独のルールセットに合わない SigmaHQ モノレポの慣例であり、その理由を
  [`detections/tests/sigma_lint/validators.yml`](detections/tests/sigma_lint/validators.yml) に記してあります。
  Suricata ルールは `suricata -T` でクリーンにロードされなければなりません。
- **再現可能なテストを併せて送ります。** ルールが発火すること(または構造が有効であること)を
  [`detections/tests/`](detections/tests/) 以下のテストで証明します。入力はバイナリをリポジトリに入れず、
  毎回決定論的に生成し、エンジンのバージョンに依存しない性質(発火の有無・誤検知なし)は厳密にアサートし、バージョンによって
  変動する数値は下限でアサートして基準値を別に記録します。Sigma 相関ルールを追加または修正すると、バックエンド
  移植性テスト([`detections/tests/sigma_backends/`](detections/tests/sigma_backends/))がそのルールが複数の
  バックエンドで変換されるかを確認するため、[`detections/README.md`](detections/README.md) のバックエンド対応の説明と
  食い違わないよう維持してください。
- **ATT&CK レイヤーも併せて更新します。** ルールに `attack.*` タグを追加または変更したら、
  [`detections/attack/artex_navigator_layer.json`](detections/attack/artex_navigator_layer.json) の技術・スコアも
  合わせて更新します。整合テストがルール↔レイヤーの双方向の一致を強制するため、レイヤーにないルールのタグや、ルールに
  ないレイヤーの技術があると失敗します。
- **攻撃の手引きと読めてしまう内容を入れません。** このリポジトリの検出資料は、防御・検出というポジショニングだけを維持します。
  エクスプロイトの実行方法や検出回避の手法のような、攻撃を助ける記述は受け付けません。

八つのテストスイートは Docker さえあればそのまま実行でき、生成物をリポジトリにコミットしません。各スクリプトは
アサーションが一つでも失敗すると 0 以外のコードで終了するため、CI や pre-commit フックにそのまま組み込めます。

```bash
detections/tests/sigma/run.sh           # Sigma: sigma check + バックエンド変換 + 指標保存
detections/tests/sigma_match/run.sh     # Sigma: 原子ルールが悪性サンプルに発火・正常サンプルに沈黙
detections/tests/sigma_lint/run.sh      # Sigma: SigmaHQ 慣例の全バリデーター + 文書化された基準
detections/tests/sigma_backends/run.sh  # Sigma 移植性: 相関ルールが複数のバックエンドで変換されるか
detections/tests/suricata/run.sh        # Suricata: pcap 合成 → suricata -r → アラート数をアサート
detections/tests/attack/run.sh          # ATT&CK: レイヤー ↔ ルールの双方向整合
detections/tests/indicators/run.sh      # 指標: ルールの固定指標 ↔ 上流ソースの双方向一致
detections/tests/misp/run.sh            # MISP: 指標 CSV ↔ MISP イベントの同期 + pymisp の有効性
```

八つを一度に実行するには、[`detections/tests/run-all.sh`](detections/tests/run-all.sh)を使ってください。CI と同じ
順序で八つを順次実行し、先行するスイートが失敗しても残りを最後まで実行した上で、スイートごとの PASS/FAIL サマリーを
出力し、一つでも失敗すれば 0 以外のコードで終了します。このランナーを pre-commit フックとしてそのまま掛ける設定例が
リポジトリルートの [`.pre-commit-config.yaml`](.pre-commit-config.yaml)にあります。`pip install pre-commit &&
pre-commit install` でインストールすると、検出ルールやそのルールが固定している上流ソースが変わるコミットでのみ(CI と同じ
範囲)ランナーが実行され、ルールとテストの不一致をプッシュ前に検知します。同じ設定ファイルには、文書内部のリンク・画像・アンカーを
検査する `docs` フック(上記の貢献手順 3 番の `check-doc-links.py`)も併せて入っています。

この八つのテストは、リポジトリ CI([`.github/workflows/detections.yml`](.github/workflows/detections.yml))が
`detections/` 以下が変更されたプッシュ・PR のたびに実行します。指標一致テストは、その指標が指す上流ソースファイル
(`enrich/`・`selfupdate/`・`guard/`・`db/`・`cmd/artex/main.go`)が変更されたときにも実行され、上流の再同期が User-Agent・
マーカー・既定ポートを変えてルールが静かに古くなる事態も併せて捕捉します。したがって、ルールだけを変更してテスト・レイヤーを更新していない変更、SigmaHQ の慣例を
破ったルール、あるいはソースと食い違うルールは、マージ前に CI で赤く表面化します。

ルールの索引と各ルールの根拠・限界は [`detections/README.md`](detections/README.md)に、テストのアサーション項目と
実行方法は [`detections/tests/README.md`](detections/tests/README.md)にまとめられています。

---

## ライセンス

このプロジェクトは **GNU Affero General Public License v3.0(AGPL-3.0)** で配布されます。
貢献物を提出すると、その貢献物も **AGPL-3.0 で公開されることに同意した**ものとみなされます。
特に、このプロジェクトを修正してネットワーク経由で(例: オンラインサービスとして)ユーザーに提供する場合は、
そのユーザーに対応する完全なソースコードを公開しなければなりません。全条項は [LICENSE](LICENSE)
ファイルにあります。
