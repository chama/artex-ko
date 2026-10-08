# 変更履歴

日本語 · [English](CHANGELOG.en.md) · [中文(原本・上流)](CHANGELOG.zh.md)

この文書は、ARTEX 韓国語版(このフォーク)が上流リポジトリに加えた変更を記録します。形式は [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) を参考にしています。

上流 ARTEX プロジェクトのバージョン別リリース履歴(0.3.x 以下)と貢献者一覧は、原本の中国語のまま [`CHANGELOG.zh.md`](CHANGELOG.zh.md) に保存してあります。上流の変更と照合しやすいよう、`README.zh.md` と同じ方式で原文をそのまま残します。各変更の詳細と根拠は、リポジトリのコミット履歴で確認できます。

## [Unreleased] · 韓国語版の変更

### ローカライズ (i18n)

- **ユーザーに公開される出力を韓国語に強制しました。** ベンチマークされたエージェントの行動指針の本文(頭脳)は性能を保つため原文のまま残し、コード固定セグメント(`langDirective`)により、ユーザーに見える成果物(脆弱性レポート、事実の要約、最終要約、チャット応答)のみを韓国語で書くよう指示します。コマンド・ペイロード・コード・ログの原文は原本を保存します。
- **Web UI を韓国語に移しました。** Next App Router に `next-intl` を導入し、文字列を `web/messages/ko.json` と `web/messages/zh.json` に分離しました。原本の中国語は `zh.json` に保存し、上流のアップデートと照合します。ダッシュボード・脆弱性・対話・通知送信・インターセプト・LLM 設定などの画面文字列を韓国語に移しました。
- **サーバー API のユーザーに公開されるエラー・応答を韓国語に移しました。** ブラウザに返される HTTP エラー・応答文言を韓国語に差し替えました。ただし、エージェントの頭脳への入力としてフィードバックされる文言は、ベンチマークのドリフトを防ぐため原文を維持し、その判定根拠はリポジトリの作業文書に記録しました。
- **文書を韓国語に整備しました。** 韓国語の `README.md` を作成して英語の `README.en.md` を併置し、原本の中国語は `README.zh.md` として保存しました。

### 防御・検出資料

- **防御・検出ガイドを追加しました。** 自律型 AI 攻撃が従来のスキャナーと何が違うのか、防御者が観測できるフィンガープリント(IoC)、エントリポイントとハードニング、検出ルール、インシデント対応をまとめた韓国語ガイド([`docs/defense-ko.md`](docs/defense-ko.md))と、同じ内容の英語版([`docs/defense-en.md`](docs/defense-en.md))を置きました。
- **配布用の検出ルールを提供します。** ガイドのフィンガープリントをすぐに使えるルールに移しました。ホスト・ログ層は [Sigma](https://sigmahq.io) の原子・相関ルール([`detections/sigma/`](detections/sigma/))、ネットワーク層は enrich プローブと norma SDK WebFetch の User-Agent を狙った [Suricata](https://suricata.io) ルール([`detections/suricata/`](detections/suricata/))として収めました。
- **ATT&CK カバレッジを可視化しました。** ルールがタグ付けする技術を MITRE ATT&CK Navigator レイヤー([`detections/attack/`](detections/attack/))にまとめました。
- **機械可読な侵害指標(IoC)を標準形式で提供します。** ARTEX が出力する固有のフィンガープリントを一つのファイルにまとめた CSV([`detections/indicators/artex_indicators.csv`](detections/indicators/artex_indicators.csv))と、同じ指標を脅威インテリジェンスプラットフォームにそのまま取り込める MISP イベント([`detections/indicators/artex_indicators.misp.json`](detections/indicators/artex_indicators.misp.json))として収めました。ルールの裏付けがある指標は `to_ids` で、ホストフォレンジックのポートは分類用の手がかりとして区別して表記します。
- **再現可能な検出テストを付けました。** ルールを実際に実行して証明するテスト八種(Sigma の構造・コンパイル検証、Sigma のリアルタイムイベントマッチング、バックエンド移植性、SigmaHQ 慣例のリント、Suricata のロード・発火、ATT&CK レイヤー整合、指標とソースの一致、MISP エクスポート ↔ CSV の同期)と、これらを一度に実行する一括ランナー・pre-commit の例を追加し、CI のマージゲートに接続しました。Sigma のリアルタイムイベントマッチングは、ルールがコンパイルされるだけでなく、悪性サンプルイベントには実際に発火し、正常なイベントには沈黙するかまで、原子・相関ルールの両方で確認します。

### リポジトリ整備

- **セキュリティ・悪用への警告と国内法の告知を入れました。** README の最上部に、使用範囲、情報通信網法・個人情報保護法の告知、悪用禁止の警告を追加しました。
- **韓国語 UI のスクリーンショットで画面プレビューを差し替えました。**
- **メンテナー向けランブックと貢献ガイドを整備しました。** 上流の同期・翻訳ドリフトを防ぐためのランブック([`MAINTAINING.md`](MAINTAINING.md))と、検出ルール貢献の契約([`CONTRIBUTING.md`](CONTRIBUTING.md))を置きました。ランブックには、リリース発行パイプラインのビルド前提と、タグなしでローカルにその前提を検証する手順も併せてまとめました。
- **プッシュ・PR マージゲートの CI を追加しました。** 上流リポジトリはタグリリースでのみ CI が動いていましたが、このフォークはすべてのプッシュと PR で、Go のビルド・静的解析(`go vet`)・単体テスト([`ci.yml`](.github/workflows/ci.yml))、韓国語 UI の静的ビルド([`web.yml`](.github/workflows/web.yml))、文書のリポジトリ内部リンク・画像参照の整合性([`docs.yml`](.github/workflows/docs.yml))を実行し、韓国語化の過程で生じたリグレッションをマージ前に捕捉します。データベースが必要な統合テストは、パッケージごとに隔離された PostgreSQL サービスで併せて検証します。文書リンクの検査は、外部ネットワークに依存しない決定論的なスクリプト([`scripts/check-doc-links.py`](scripts/check-doc-links.py))で実行し、多言語文書が互いを指す多くの相対リンクと画面プレビュー画像が壊れたままマージされることを防ぎます。文書アンカー(`#見出し`)リンクも GitHub と同じ slug 規則で見出しと照合し、見出しの文字が変わって静かに切れた目次・相互参照リンクも併せて捕捉します。検出ルールスイートは、上記「防御・検出資料」の節で説明したマージゲートが担当します。
- **外部リンクの生存を定期的に点検します。** 防御ガイドが指すインシデント通報窓口・標準参照のような外部リンクは、リモートサーバーの状態に依存して flaky になるため、マージゲートから外し、非ブロッキングのワークフロー([`external-links`](.github/workflows/external-links.yml))が毎週月曜と手動実行で、ブラウザ User-Agent・GET・リダイレクト追跡による点検([`scripts/check-external-links.py`](scripts/check-external-links.py))を実行します。ホストは生きているのに確認方法だけが塞がれている場合(ボット遮断・レート制限)と、私たちが直せない上流継承のリンク切れ(allowlist)は失敗として扱わず、私たちの文書がキュレーションした外部リンクが新たに壊れたときにのみ赤く表面化させます。
- **貢献・ガバナンスのインフラを整えました。** バグ・機能・翻訳の Issue テンプレート([`.github/ISSUE_TEMPLATE/`](.github/ISSUE_TEMPLATE/))とプルリクエストテンプレート([`PULL_REQUEST_TEMPLATE.md`](.github/PULL_REQUEST_TEMPLATE.md))、セキュリティ脆弱性の報告ポリシー([`SECURITY.md`](SECURITY.md))、行動規範([`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md))を置き、外部の貢献者が Issue・PR・セキュリティ報告を一貫した様式で提出できるようにしました。
- **海外の貢献者向けの英語文書レイヤーを完成させました。** このリポジトリは韓国語が主言語ですが、韓国語を読めない貢献者・セキュリティ研究者・防御者が同じ情報に到達できるよう、主要文書の英語版も併せて置きました。英語の `README.en.md`・防御ガイド([`docs/defense-en.md`](docs/defense-en.md))に加えて、変更履歴([`CHANGELOG.en.md`](CHANGELOG.en.md))、セキュリティ報告ポリシー([`SECURITY.en.md`](SECURITY.en.md))、行動規範([`CODE_OF_CONDUCT.en.md`](CODE_OF_CONDUCT.en.md))、貢献ガイド([`CONTRIBUTING.en.md`](CONTRIBUTING.en.md))、メンテナー向けランブック([`MAINTAINING.en.md`](MAINTAINING.en.md))、トラフィック証拠の設計文書([`docs/finding-traffic-evidence-en.md`](docs/finding-traffic-evidence-en.md))、そしてバグ・機能・翻訳の Issue テンプレートの英語版を揃えました。韓国語版と英語版は冒頭でお互いを指しており、どの言語から入っても反対側へ移動できます。(プルリクエストテンプレートは現在、韓国語版のみ提供しています。)

---

上流 ARTEX プロジェクトのバージョン別リリース履歴と貢献者一覧は、[`CHANGELOG.zh.md`](CHANGELOG.zh.md) で原文のままご覧いただけます。
