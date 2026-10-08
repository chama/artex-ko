# ARTEX ATT&CK カバレッジ

日本語 · [English](README.md)

このリポジトリの検知ルールがタグ付けしている [MITRE ATT&CK](https://attack.mitre.org/)(Enterprise)の技法を
Navigator レイヤーとしてまとめたものです。[Sigma ルール](../sigma/)の `attack.*` タグから手作業で作成しており、
すべての技法は、指標がこのリポジトリのソースで確認した文字列や挙動であるルールに基づいています。推測で加えた
項目はなく、[整合性テスト](../tests/attack/run.sh)がレイヤーとルールの食い違いを防ぎます。

- **`artex_navigator_layer.json`**: ATT&CK Navigator v4.5 形式のレイヤーです。

## スコアの意味

ここでのカバレッジは「このリポジトリがこの技法をタグ付けする検知を提供している」という意味であり、「この技法が
完全にカバーされている」という意味ではありません。スコアは検知の強度を意図的に正直に付けています。

- **100: ARTEX 固有のシグネチャまたは挙動。** ARTEX にしかない静的指標(`artex-enrich/1.0`・
  `artex-selfupdate` User-Agent、ガード監査マーカー)であるか、その上に構築した行動ルール(エンリッチ速度・ファンアウト、
  ガードブロックのバースト)です。
- **50–65: 汎用的なハンティングの手がかり。** ARTEX ガードのブロックリストを反映した破壊的コマンドのハンティングです。
  同じコマンドは正当な管理者も実行するため、良性(benign)の活動でも発火します。ヒットは手がかりとして扱い、断定の
  根拠にはしないでください。65 は、相関ルールがそのコマンドを ARTEX ガードマーカーと組み合わせて特異度を高めた
  場合を指します。

## 対象とする技法

6 つの戦術にまたがる 8 つの技法です。各技法は、それをタグ付けするルールに対応します。

- **偵察(Reconnaissance): T1595 (Active Scanning), T1592 (Gather Victim Host Information)。**
  [`sigma/artex_enrich_user_agent.yml`](../sigma/artex_enrich_user_agent.yml)、
  [`sigma/correlation/artex_enrich_scan_velocity.yml`](../sigma/correlation/artex_enrich_scan_velocity.yml)、
  [`sigma/correlation/artex_enrich_fanout.yml`](../sigma/correlation/artex_enrich_fanout.yml)、そして
  [Suricata ルール](../suricata/artex.rules)(sid 1000001 / 1000002)です。
- **コマンド&コントロール(Command and Control): T1105 (Ingress Tool Transfer)。**
  [`sigma/artex_selfupdate_egress.yml`](../sigma/artex_selfupdate_egress.yml)です。
- **実行(Execution): T1059 (Command and Scripting Interpreter)。**
  [`sigma/artex_guard_audit_framing.yml`](../sigma/artex_guard_audit_framing.yml)、
  [`sigma/correlation/artex_guard_block_burst.yml`](../sigma/correlation/artex_guard_block_burst.yml)、
  [`sigma/correlation/artex_guard_marker_then_destructive.yml`](../sigma/correlation/artex_guard_marker_then_destructive.yml)です。
- **インパクト(Impact): T1485 (Data Destruction), T1561.002 (Disk Wipe: Disk Structure Wipe), T1489 (Service Stop)。**
  [`sigma/destructive_command_hunting.yml`](../sigma/destructive_command_hunting.yml)であり、T1485 は
  [`sigma/correlation/artex_guard_marker_then_destructive.yml`](../sigma/correlation/artex_guard_marker_then_destructive.yml)でも補強されます。
- **認証情報アクセス・収集(Credential Access / Collection): T1557 (Adversary-in-the-Middle)。**
  [`sigma/artex_recording_proxy_ca.yml`](../sigma/artex_recording_proxy_ca.yml)であり、ワーカーツールの通信を
  復号・記録するために ARTEX 内蔵のトラフィックレコーダー(`traffic/traffic.go`)がインストールする MITM ルート CA アーティファクトを
  狙った、ホスト・フォレンジック向けのハンティングの手がかりです。

## 使い方

1. [ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/)を開きます。
2. **Open Existing Layer → Upload from local** を選び、`artex_navigator_layer.json` を選択します
   (または、このリポジトリの raw ファイル URL を指定します)。
3. スコアが付いた技法が検知強度に応じて色分けされて表示され、各技法には根拠となったルールファイルと
   防御ガイドの節を記した注釈が付いています。

## 範囲と正直さ

- **カバレッジは網羅性ではありません。** ここでスコアを得た技法は、ルールがそれをタグ付けしているという意味であり、その
  技法のあらゆる変種を検知するという意味ではありません。ネットワーク上で ARTEX 固有の User-Agent によって捕捉できるシグナルは、
  偵察段階のエンリッチプローバー(`artex-enrich/1.0`)と、攻撃段階の norma SDK WebFetch(`norma/0.4`)の 2 つだけで、
  それ以外の攻撃トラフィックはツール既定のフィンガープリントに従います。長持ちする検知は行動ベースです
  (防御ガイド [韓国語](../../docs/defense-ko.md) · [English](../../docs/defense-en.md) の 1~2 節・4.1~4.2 節を参照)。純粋な Web 多段階の事例には、
  依然として環境ごとのベースラインルールが必要です。
- **静的指標は変更できます。** オペレーターが User-Agent を別の値に設定できるため、タグ付けされた
  指標がないからといって安全とは限りません。ルールファイルにも同じ注意書きを付けています。

## 検証と貢献

[整合性テスト](../tests/attack/run.sh)を実行してください。Docker さえあれば動作し、レイヤーがスコアを付けた
技法・戦術がルールの `attack.*` タグと正確に一致していること、そしてすべての技法が実在するルールファイルに
基づいていることをアサートします。

```sh
detections/tests/attack/run.sh
```

ルールを追加したり再タグ付けしたりした場合は、このレイヤーもそれに合わせて更新してください。ルールの技法がレイヤーにない場合や、
レイヤーの技法がルールにない場合、テストは失敗します。[`../README.ko.md`](../README.ko.md)と
[`../../CONTRIBUTING.md`](../../CONTRIBUTING.md)を参照してください。
