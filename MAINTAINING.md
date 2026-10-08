# 上流との同期と翻訳ドリフトの防止（メンテナー向けガイド）

한국어 · [English](MAINTAINING.en.md)

このドキュメントは、**メンテナー**が元リポジトリ [Autumn-27/ARTEX](https://github.com/Autumn-27/ARTEX) の
変更に追随しながら、韓国語ローカライズを維持する手順をまとめたものです。貢献の範囲・法的責任・
ローカライズ方針は [CONTRIBUTING.md](CONTRIBUTING.md) に、ユーザー向けの案内は [README.md](README.md) に
ありますので、このドキュメントではその方針を**実際にどう運用するか**だけに絞って説明します。

ローカライズの核となる目標は、一文で要約できます。元の**判断性能をそのまま保ちつつ、ユーザーに
見える成果物だけを韓国語に置き換える**ことです。上流が更新されるたびにこの境界は崩れやすいため、
以下の手順と検査で翻訳ドリフトを防ぎます。

---

## 1. ローカライズ構造の概要

このリポジトリは、上流の ARTEX を**フォーク**し、その履歴の上に韓国語ローカライズのコミットを
積み上げた構造です。上流 `main` のすべてのコミットがこのリポジトリの履歴に含まれており、その上に
ローカライズのコミットが追加されています。したがって、上流の変更を取り込む作業は、「上流 `main` との
差分を確認し、保存するものと翻訳するものを仕分けて反映すること」になります。

成果物は三つに分かれます。

- **原文のまま残す資産**（下記の 2 節）。翻訳すると性能や上流との突き合わせが壊れます。
- **コードに固定された出力言語の強制**。`agent/prompt.go` の `langDirective()` が、各ロールの system
  プロンプトの末尾に「ユーザーに見える出力は韓国語で書くこと」という指示を付け足します。
- **韓国語に翻訳するユーザー向け文字列**。UI は `web/messages/ko.json` に、サーバーの
  ユーザー向け応答文言は各 Go ファイルの名前付き定数に置きます。

---

## 2. 原文を保存する資産（翻訳禁止）

次の資産は翻訳せず、原文（中国語または英語）を維持します。上流の変更がこれらの資産に及んだ場合は、
**翻訳せずそのまま反映**します。

- **エージェントの内部推論プロンプト（頭脳本体）。** `agent/promptcatalog.go` と DB シード
  `agent_prompts` にある行動指針の本文です。原文（中国語）でベンチマークされた動作を維持する
  必要があるため、翻訳すると判断にドリフトが生じます。
- **表示とエージェント入力を兼ねる文字列。** アクティビティタイムラインに表示されると同時に、
  プランナー・リポーターの入力コンテキストとして再投入される一部の文言（作業中断の理由、
  インターセプトのブロックメッセージ、トラフィック証拠ヘルパーなど）は、一つのレコードが二つの
  用途を兼ねるため、原文を保存します。判定根拠は `work/DECISIONS-FOR-JIWOO.md` の
  「頭脳境界の記録」に案件ごとに記載されています。
- **元の中国語ドキュメント・文字列。** ドキュメントは `README.zh.md`、UI 文字列は
  `web/messages/zh.json` に原文をそのまま残し、上流の変更と突き合わせやすくします。韓国語訳は
  `web/messages/ko.json` にのみ入れます。
- **コマンド・ペイロード・コード・URL・識別子・ログの原文。** 分析に必要な元データなので翻訳しません。
  Go コードのコメントも優先順位が最も低いため、上流との突き合わせが終わる時点まで原文のままにします。

---

## 3. 上流追跡のベース

上流リモートが次のように設定されている必要があります。なければ追加します。

```bash
git remote add upstream https://github.com/Autumn-27/ARTEX
git remote -v   # upstream が表示されることを確認
```

現在、ローカライズが反映を完了している上流のベースコミットは次のとおりです。

- **ベース = `d003372`**（上流 `main`、2026-10-03、PR #189 `fix/sse-same-origin` のマージ）。

この値は「このコミットまでの上流の変更はすべてこのリポジトリに取り込まれている」という意味です。
上流の変更を新たに反映するたびに、このベースを 7 節の方法で更新します。

---

## 4. 上流の変更を取り込む手順

### 4.1 上流を取得して差分を確認します

```bash
git fetch upstream
git rev-list --count d003372..upstream/main        # 未反映の上流コミット数
git log --oneline d003372..upstream/main           # 未反映コミットの一覧
```

`git fetch` は上流のリモート追跡ブランチだけを更新するため、作業ツリーと `HEAD` には影響しません。
未反映のコミットが 0 であれば上流と同期済みの状態なので、これ以上やることはありません。

### 4.2 変更ファイルを分類します

未反映のコミットがどのファイルに触れたかを確認し、2 節の保存資産と翻訳対象に分けます。

```bash
git log --name-status --oneline d003372..upstream/main
```

分類基準は次のとおりです。

- `agent/promptcatalog.go`・`agent_prompts` シード、および 2 節の「表示兼入力」文字列が変更された場合
  → **翻訳せずそのまま反映**します。
- Go バックエンドのロジック（`db/`・`llmrec/`・`server/` など）が変更された場合 → ロジックはそのまま反映しつつ、
  **新しく追加されたユーザー向け応答文言**（`writeErr` など）がないかを確認し、あれば韓国語の定数に翻訳します。
- UI（`web/src/**`）が変更されて**新しい画面文字列**が増えた場合 → ハードコードせず、
  `web/messages/zh.json`（原文）と `web/messages/ko.json`（翻訳）に同じキーで追加します。
- **検知ルールが固定している上流の指標**（`enrich/enrich.go` のプローバーの User-Agent、`selfupdate/` の
  自己更新 User-Agent、`guard/guard.go` の監査マーカー、`db/db.go` の破壊的コマンド deny リスト、
  `cmd/artex/main.go` のデフォルトのリッスン・記録プロキシのポート）が変更された場合 → `detections/` の
  Sigma・Suricata ルールと ATT&CK レイヤー、さらに `detections/indicators/artex_indicators.csv` の
  値も新しい値に合わせます。この指標は翻訳対象ではなく**検知の根拠**であるため、上流が値を変えると
  ルールが静かに古くなります。5.4 の指標一致テストがこのずれを自動的に検出します。

### 4.3 反映します

機能単位でマージするか選別して反映したうえで、4.2 で洗い出した新しい文字列を韓国語に翻訳します。
マージの過程で `ko.json`・`zh.json` のキーがずれたり、ユーザーに見える箇所に原文が混入したりしやすいため、
反映の直後に必ず 5 節の検査を実行します。

> **例（2026-10-05 時点の未反映コミット）。** `git fetch upstream` の結果、上流 `main` が
> `b55ceb1` まで進んでおり、ベース `d003372` と比べて 2 つのコミット（`86729b6` モデルフォールバック承認トークン計量
> 機能 + マージコミット `b55ceb1`）が未反映です。このコミットは `db/llm_usage.go`・`llmrec/llmrec.go`・
> `server/intercept.go`・`server/server.go` といった Go ロジックと `web/src/app/(main)/system/intercept/page.tsx`・
> `web/src/lib/api.ts`・`web/src/lib/mock/handler.ts`・`web/src/lib/types.ts` に触れています。
> そのためメンテナーは、Go ロジックはそのまま反映し、intercept 設定ページに新しく追加された画面
> 文字列だけを `ko.json`・`zh.json` のキーとして抽出・翻訳すれば済みます。（この 2 つのコミットは、このドキュメントを書いた時点では
> まだ反映していないため、ベースは `d003372` のままにしてあります。）

---

## 5. 翻訳の対称性とドリフトの検査

上流の反映や翻訳作業の後に、次の三つを確認します。

### 5.1 ko ↔ zh メッセージの対称性とユーザー向け CJK

`ko.json` と `zh.json` のキーがまったく同じで、`ko.json` の値に中国語の漢字が残っていてはいけません。
次のスクリプトが三つの数値を出力します。

```bash
python3 - <<'PY'
import json, re
ko = json.load(open('web/messages/ko.json'))
zh = json.load(open('web/messages/zh.json'))
def flatten(d, p=''):
    out = {}
    if isinstance(d, dict):
        for k, v in d.items(): out.update(flatten(v, p + '/' + k))
    elif isinstance(d, list):
        for i, v in enumerate(d): out.update(flatten(v, p + '/' + str(i)))
    else: out[p] = d
    return out
fk, fz = flatten(ko), flatten(zh)
han = re.compile(r'[㐀-鿿]')
print('ko leaf keys :', len(fk))
print('zh leaf keys :', len(fz))
print('key symdiff  :', len(set(fk) ^ set(fz)))        # 0 であること
print('ko vals w/CJK:', sum(1 for v in fk.values() if isinstance(v, str) and han.search(v)))  # 0 であること
PY
```

基準値（2026-10-05）：`ko leaf keys = 2950`、`zh leaf keys = 2950`、`key symdiff = 0`、
`ko vals w/CJK = 0`。キー数は上流の反映で増えることがありますが、ko と zh は常に同じでなければならず、
`key symdiff` と `ko vals w/CJK` は常に 0 でなければなりません。

### 5.2 頭脳資産の原文保存の確認

頭脳本体は中国語の原文を維持するため、次の検査で**漢字を含む行数が 0 に落ちた場合**は、むしろ
頭脳が誤って翻訳されて汚染されたというシグナルです。

```bash
python3 -c "import re; han=re.compile(r'[㐀-鿿]'); t=open('agent/promptcatalog.go').read(); print('promptcatalog.go CJK lines =', sum(1 for l in t.splitlines() if han.search(l)))"
```

基準値（2026-10-05）：`promptcatalog.go CJK lines = 70`。この数が大きく減った場合は、頭脳本体が翻訳されて
いないかを確認します。

### 5.3 ビルド成果物に原文が漏れていないか

UI を静的にエクスポートした後、プリレンダリングされた HTML に中国語が見えたら翻訳漏れです。

```bash
cd web && npm ci && NEXT_EXPORT=1 npm run build   # out/ を生成
# out/**/*.html の可視テキストに中国語の漢字が 0 件であることを確認
```

### 5.4 検知指標が上流ソースと今も一致しているか

`detections/` のルールは、上流が実際に出力する文字列（プローバーの User-Agent・自己更新
User-Agent・監査マーカー・破壊的コマンドの deny リスト）に基づいています。上流の再同期でこの値が変わると、
翻訳の検査はすべて通るのに、デプロイ済みのルールだけが静かにマッチしなくなります。次のテストは、各指標が
上流ソースとルールの両方に今もあることを双方向で確認しますので、再同期の後に併せて実行します。

```bash
detections/tests/indicators/run.sh   # Docker で隔離実行。RESULT: PASS なら一致
```

失敗した場合は、どの指標がずれたか、およびその方向（上流ソースが変わったのか、ルールが変わったのか）が
出力されるので、4.2 の最後の分類基準のとおりにルール・レイヤーを新しい値に合わせます。このテストはリポジトリの
CI（[`.github/workflows/detections.yml`](.github/workflows/detections.yml)）でも、ルールツリーや
上記の上流ソースファイルが変更されたプッシュ・PR のたびに自動で実行され、再同期ドリフトをマージゲートで検出します。

新しい指標を追加する際に**新しい上流ソースファイルを固定した場合**（例：`cmd/artex/main.go` のポート指標を
追加したときのように）、そのファイルを必ず上記ワークフローの `push`・`pull_request` の `paths` フィルターにも追加します。
追加し忘れると、そのソースだけを変更した PR は指標テストを起動できず、ドリフトがマージゲートを静かに
すり抜けます。この同期自体も指標テストが自動的に確認します（5 つ目の検査「CI triggers this
test when any pinned source changes」）：テストが読み込む `detections/` 以外のすべてのソースが両方の `paths`
ブロックに列挙されていなければテストが失敗するため、ソースの固定と CI の起動条件がずれたままマージ
されることはありません。

### 5.5 検知テストツールのピンを上げるとき

検知テストは `sigma-cli`・SigmaHQ バリデータープラグイン（`pySigma-validators-sigmahq`）・Suricata イメージを
固定バージョンで実行します（各 `run.sh` のデフォルト値で、環境変数で上書き可能）。このピンを上げると、上流の
ソースではなく**ツール側のドリフト**が発生することがあります。特に SigmaHQ バリデーターはバージョンアップのたびに新しい慣例
チェックを追加するため、`detections/tests/sigma_lint/run.sh` が新しい問題を赤く表示することがあります。
その場合は、ルールを新しい慣例に合わせるか、単独のルールセットには合わない慣例であれば、その理由を記載して
[`detections/tests/sigma_lint/validators.yml`](detections/tests/sigma_lint/validators.yml) の除外
リストに追加します。バックエンドプラグインがサポート内容を変更した場合は、`sigma_backends` テストが同じシグナルを出します。

---

## 6. ビルドとテストによる最終検証

反映・翻訳の後は、[CONTRIBUTING.md の開発環境](CONTRIBUTING.md#開発環境)の手順に従ってバックエンドと
フロントエンドを検証します。ローカルに Go がなければ、Docker で同じように実行できます。

```bash
docker run --rm -v "$PWD":/src -w /src \
  -v artexko-gomod:/go/pkg/mod -v artexko-gocache:/root/.cache/go-build \
  golang:1.26 sh -c 'go build ./... && go vet ./... && go test ./... -count=1'
```

ユーザー向けの文言を翻訳するときは、その文言をアサートする回帰テスト（`*_localized_test.go`）を併せて
置き、後で上流の変更が再び中国語を持ち込んでもテストが検出できるようにします。翻訳の検証は必ず
能力の高い（フロンティア級の）モデルで行います。低価格・小型のモデルは出力が原文に戻ってしまうことがあり、
翻訳が適用されているかどうかをその出力だけで判断してはいけません。

---

## 7. ベース更新の記録

上流の変更を反映して検証まで終えたら、このドキュメントの 3 節にある**ベースコミットの値を新しい上流コミットに
更新**し、その変更を同じコミットまたは後続のコミットに含めます。こうしておけば、次のメンテナーが
「どこまで反映されたか」をこのドキュメント一か所で確認できます。

コミットメッセージは [CONTRIBUTING.md のコミットメッセージ規則](CONTRIBUTING.md#コミットメッセージ)に従います。
たとえば、上流同期のコミットは次のように書きます。

```
chore(upstream): 上流 d003372..b55ceb1 を反映 (intercept トークン計量) + 新規 UI 文字列の翻訳
```

---

## 8. レビュー・検証の心得（よくある落とし穴）

上流の反映・翻訳・ドキュメント補強を点検するときに、メンテナーが繰り返しはまる落とし穴を二つ記しておきます。
どちらも「検査方法そのものが間違っているために、問題のないものを壊れていると誤認する」ケースなので、
不要な差し戻しを防ぐために心得として固定します。

### 8.1 リポジトリの CI 状態はリポジトリを指定して確認します

このリポジトリは上流 ARTEX のフォークであるため、ローカルの `git remote` には `origin`（jiwoochris/artex-ko）と
`upstream`（Autumn-27/ARTEX）が併せて登録されています（3 節を参照）。この状態で `gh` コマンドに
リポジトリを指定しないと、`gh` が**上流リポジトリをデフォルトとして選び**、当方のワークフローがない
上流の実行結果を表示します。すると、上流の CI が緑であるのを見て**当方の CI が通ったと
勘違いしたり**、当方のワークフロー（`ci.yml`・`detections.yml`）を「HTTP 404 … not found」と誤って
判断したりすることがあります。

そのため CI を確認するときは、常にリポジトリを明示します。

```bash
gh run list -R jiwoochris/artex-ko --workflow ci.yml --limit 5
gh run list -R jiwoochris/artex-ko --workflow detections.yml --limit 5
```

一度設定しておけば、`-R` を省略しても当方のリポジトリをデフォルトで参照するように変更できます。ただしこの
設定は**ローカルの gh 設定**であり、リポジトリにはコミットされないため、新しいマシンや新しいチェックアウトでは再度
指定する必要があります。

```bash
gh repo set-default jiwoochris/artex-ko
gh repo set-default --view   # jiwoochris/artex-ko が表示されることを確認
```

### 8.2 ドキュメントの外部リンクはブラウザと同じ GET で確認します

防御ガイド（[`docs/defense-ko.md`](docs/defense-ko.md)・[`defense-en.md`](docs/defense-en.md)）の
7 節は、国内の公式チャネル（boho.or.kr・fsec.or.kr・pipc.go.kr）のリンクを掲載しています。このリンクが生きているか
確認するときに、`curl -I`（HEAD リクエスト）やデフォルトの User-Agent だけで確認すると、**問題のないリンクを壊れて
いると誤認**します。国内の公的機関・セキュリティ機関のサイトは、次の三つの理由で単純な確認を拒否するためです。

- **HEAD リクエストを拒否します。** たとえば fsec.or.kr は `curl -I`（HEAD）に対して 400 を返します。
- **デフォルトの `curl` User-Agent をブロックします。** fsec.or.kr と pipc.go.kr は、デフォルトの UA で送った
  GET リクエストにも 400 を返します（ブラウザの UA で送ると 200）。
- **別のアドレスにリダイレクトします。** pipc.go.kr は `www.pipc.go.kr` から `pipc.go.kr/np/` へ
  2 回リダイレクトするため、リダイレクトを追跡しないと最終的な状態を見逃します。

したがって、リンクの確認は**ブラウザの User-Agent で、GET で、リダイレクトを追跡しながら**行います。

```bash
UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0 Safari/537.36'
for u in https://www.boho.or.kr https://www.fsec.or.kr https://www.pipc.go.kr; do
  curl -sS -L -A "$UA" -o /dev/null -w "$u -> %{http_code} %{url_effective}\n" "$u"
done
```

最終的なステータスコードが 200 であれば、リンクは有効です。ステータスコードが 400・403 になった場合は、リンクが壊れた
のではなく、**確認方法がサーバーのアクセスポリシーに阻まれたのではないか**をまず疑い、HEAD・デフォルトの
UA・リダイレクト未追跡といった要因を一つずつ取り除いて再確認します。（2026-10-06 の確認時点では、
三つのリンクすべてが上記の方法で 200 であり、pipc.go.kr は 2 回のリダイレクトの後に 200 です。）

この手動の手順は、`scripts/check-external-links.py` がそのまま自動化しています。追跡対象のすべての `.md`
からコードフェンス・インラインコードの外にある外部リンクを集め（予約・プレースホルダーのホストは除外）、上記と
同様にブラウザの UA・GET・リダイレクト追跡で状態を確認し、ネットワークエラー・5xx・429 は再試行して
一時的な揺らぎと本当の障害を見分けます。結果を四つに分けます：OK（2xx・3xx）・
RESTRICTED（401・403・405・429。ホストは生きており、確認方法だけが阻まれている）・ALLOWED
（`scripts/external-links-allowlist.txt` に記載された、当方では直せない上流由来の死んだリンク）・
DOWN（404・410・5xx・接続エラー。壊れている可能性が高い）。

- ネットワークなしで点検対象だけを事前に確認：`python3 -I scripts/check-external-links.py --list`
- リリース・定期点検（新たに壊れたリンクがあれば異常終了）：`python3 -I scripts/check-external-links.py --strict`

外部リンクの生存確認は不安定なため、**マージゲートには入れません**。その代わり、非ブロッキングのワークフロー
[`external-links`](.github/workflows/external-links.yml) が毎週月曜日と手動実行で
`--strict` を実行し、allowlist にない DOWN が新たに発生すると赤く表示します。上流の原文保存
ファイルが引き継いだ死んだリンク（例：`CHANGELOG.zh.md` がクレジットしている、消えた貢献者アカウント）は、当方では
直せないため、allowlist に理由を添えて記載し、strict の点検から除外します。

---

## 9. リリース発行パイプライン

バージョンタグ（`v*`）をプッシュすると、[`.github/workflows/release.yml`](.github/workflows/release.yml) が 5 つの
プラットフォーム向けバイナリと、条件を満たす場合はマルチアーキテクチャの Docker イメージを作成します。このフォークは、まだ
リリースタグを切ったことがなく、このワークフローが一度も実行されていないため、この節ではパイプラインが
何を前提とし、何を生成するのか、そしてその前提が現在のリポジトリ構造と合っているかを整理します。
タグをプッシュすると公開リポジトリに GitHub Release が作成されるため、リリースを切る作業は、発行の判断が固まった後に
行います。

### 9.1 リリースを切る方法

`v` で始まるタグをプッシュすると、ワークフローが起動します。

```bash
git tag v0.3.15
git push origin v0.3.15
```

### 9.2 パイプラインが行うこと

ワークフローは 5 つのジョブに分かれています。

- **frontend.** フロントエンドを静的に一度エクスポートし（`web/out`）、その成果物を `web-dist`
  アーティファクトとしてアップロードします。後続の binaries ジョブが、ターゲットごとにこの成果物を再度受け取って再利用します。
- **binaries.** 5 つのターゲット（linux amd64・arm64、darwin amd64・arm64、windows amd64）をクロス
  コンパイルし、ターゲットごとに zip にまとめます。linux amd64 のバイナリには `artex -h` のスモークテストを
  実行し、バイナリが実際に動作するかを確認します。
- **release.** すべての zip を集めて `SHA256SUMS` チェックサムを作成し、GitHub Release を作成して zip と
  チェックサムを添付します。
- **docker-gate.** `DOCKERHUB_USERNAME`・`DOCKERHUB_TOKEN` シークレットが設定されているかを確認し、その
  結果を次のジョブの実行条件として渡します。
- **docker.** 上記のシークレットがあるときだけ実行され、binaries がクロスコンパイルした linux バイナリを受け取って
  マルチアーキテクチャイメージをビルドし、Docker Hub にアップロードします。シークレットがなければこのジョブをスキップするため、
  リリース CI は赤い失敗なしに、バイナリのリリースだけで完了します。

### 9.3 ビルドの前提がリポジトリ構造と合っているか

パイプラインは次の三つの前提の上で動作します。この前提が現在のリポジトリ構造とすべて合っているかを、
ローカルで binaries ジョブを直接再現して確認しました。

- **フロントエンドの埋め込み。** frontend ジョブがアップロードした `web-dist`（= `web/out` の内容）を binaries ジョブが
  `server/webui/dist` として受け取り、`server/webui_embed.go` の `//go:embed all:webui/dist` がその場所を
  バイナリに埋め込みます。そのため binaries ジョブは、フロントエンドを再ビルドせず
  `ARTEX_SKIP_FRONTEND=1` で [`build.sh`](build.sh) を呼び出します。
- **バイナリ・パッケージのパス。** `build.sh --target <os>/<arch>` は、`dist/artex-<os>-<arch>/artex`
  バイナリと、`dist/` 配下の zip パッケージを作成します。zip にはバイナリと併せて起動スクリプト（Linux・
  macOS は `start.sh`、Windows は `start.bat`）、`skills/`、`config.example.json`、`README.md` が
  入ります。
- **Docker イメージへのバイナリのコピー。** binaries ジョブは linux バイナリを `bin-linux-<arch>`
  アーティファクトとして別にアップロードし、docker ジョブがこれを `dist/<arch>/artex` として受け取ります。
  [`Dockerfile`](Dockerfile) の `COPY dist/${TARGETARCH}/artex` が、マルチアーキテクチャビルドで buildx
  が各プラットフォームに合わせて埋めてくれる `TARGETARCH` によって、そのパスを指します。
  [`.dockerignore`](.dockerignore) は `dist/` を除外していないため、バイナリがビルドコンテキストに
  含まれます。

### 9.4 まだ決定前のこと：Docker イメージの名前空間

docker ジョブは現在、イメージ名を上流の `autumn27/artex` にしており、このフォークをどの
名前空間で発行するかは別途決定すべき事項です（`work/DECISIONS-FOR-JIWOO.md` の 8 番の項目）。決定が
下るまでは Docker Hub のシークレットを置かず、その間のリリースはバイナリの zip とチェックサムだけを
発行します（docker ジョブはスキップされます）。

### 9.5 タグなしでローカルで事前検証する

公開リリースを切らずにパイプラインの前提だけを確認するには、binaries ジョブをローカルで再現します。
ローカルに Go がなければ、Docker で同じように実行できます。

```bash
# 1) フロントエンドの静的エクスポート (release.yml の frontend ジョブに相当)
cd web && npm ci && npm run build:static && cd ..
# 2) binaries ジョブがアーティファクトを受け取る場所に配置
rm -rf server/webui/dist && mkdir -p server/webui/dist && cp -a web/out/. server/webui/dist/
# 3) 1 つのターゲットだけを binaries ジョブと同じ環境でビルド
docker run --rm -v "$PWD":/app -w /app \
  -e ARTEX_SKIP_FRONTEND=1 -e ARTEX_SKIP_NPM_CI=1 \
  -e ARTEX_COMPRESS=0 -e ARTEX_PACKAGE=1 -e ARTEX_PACKAGE_DIR=dist \
  -e ARTEX_BUILD_VERSION=v0.0.0-local \
  golang:1.26 bash -c 'apt-get update && apt-get install -y zip && ./build.sh --target linux/amd64'
# 4) 成果物を確認: dist/artex-linux-amd64/artex · dist/*.zip · dist/SHA256SUMS
```

`dist/artex-linux-amd64/artex` は静的リンクされた ELF で、`-h` を付けると使い方を出力した後、終了
コード 0 で終わります。これが binaries ジョブのスモークテストで確認される動作です。ビルド成果物
（`dist/`・`server/webui/dist/`）はリポジトリにコミットしません（`.gitignore` で除外されます）。
