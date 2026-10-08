# ARTEX 検知ルール (Suricata / ネットワーク)

日本語 · [English](README.md)

[Sigma ルール](../sigma/)のネットワーク層における対になるものです。この [Suricata](https://suricata.io) シグネチャは、
ネットワーク上で観測される 2 種類の ARTEX 成果物を対象とし、すべての指標は推測ではなく、このリポジトリの
ソースで確認した文字列や挙動に基づいています。ホスト・ログ・SIEM 層は [`../sigma/`](../sigma/)にあり、
全体像は防御ガイド([韓国語](../../docs/defense-ko.md) · [English](../../docs/defense-en.md))が
説明しています。

## ルール: [`artex.rules`](artex.rules)

- **sid 1000001**: `ARTEX enrichment prober User-Agent`。User-Agent が `artex-enrich/` で始まる
  インバウンド HTTP `GET` です(アセットエンリッチプローバー `enrich/enrich.go:233`)。単発リクエストの存在指標です。
  `classtype: attempted-recon`。
- **sid 1000002**: `ARTEX enrichment prober high-rate enumeration`。同じ User-Agent が
  `detection_filter` のしきい値である **送信元あたり 300 秒間に 30 リクエスト** を超える場合です。単発ルールが見逃す、機械
  速度で大量に送られる頻度を捉えます。Sigma 相関ルール `artex_enrich_scan_velocity` に対応します。
  `classtype: attempted-recon`。
- **sid 1000003**: `ARTEX worker WebFetch User-Agent`。User-Agent が `norma/` で始まるインバウンド
  HTTP リクエストです(norma SDK の WebFetch ツール `github.com/Autumn-27/norma/tool/webfetch.go`)。この UA は norma の全バージョン
  (v0.1.0–v0.4.3、検証済み)にわたってハードコードされており、記録プロキシがリクエストヘッダーを変更
  しないため(`traffic/traffic.go`)、対象ホストのワイヤにそのまま到達します。エンリッチプローバーと異なり、
  **攻撃段階**(能動的な脆弱性プロービング)で発火します。`classtype: attempted-recon`。

## 範囲と正直さ: デプロイ前にお読みください

- **2 種類の ARTEX User-Agent がネットワークで観測されます。** エンリッチプローバーは偵察段階で
  `artex-enrich/1.0`(`enrich/enrich.go:233`)を、norma SDK の WebFetch ツールは攻撃段階で
  `norma/0.4`(`github.com/Autumn-27/norma/tool/webfetch.go`)を送信します。記録プロキシ(`traffic/traffic.go`)はリクエストヘッダーを
  変更しないため、どちらの UA も対象のワイヤに到達します。それ以外の worker ツール(Bash サブプロセスである
  `curl`、`nmap` など)は独自の User-Agent を使用するため、一般的なスキャナーシグネチャと
  [`../sigma/`](../sigma/)の行動ベースの SIEM ルールで検知してください。
- **User-Agent は平文でのみ見えます。** 通信が平文 HTTP であるか、TLS を終端するプロキシ・WAF で
  検査される場合に現れます。エンドツーエンドの TLS はこれを暗号化するため、実際に HTTP リクエストバッファを見られる
  位置にデプロイしてください。
- **静的な User-Agent はオペレーターが変更できるため**、ないからといって安全という意味では**ありません**。
  長持ちするシグナルは行動、すなわち速度と幅です。sid 1000002(および Sigma 相関層)が速度を基準に
  している理由、そして純粋な Web 多段階の検知が環境ごとのベースラインルールを必要とする理由がここにあります。
- **意図的に除外したもの。** 自己アップデート User-Agent `artex-selfupdate` は GitHub への HTTPS に乗るため、
  ネットワークでは観測されません(TLS SNI だけではアラートを上げるには一般的すぎます)。監査統制マーカーは
  対象に向かう通信ではなくオペレーター側のログ成果物であるため、
  [`../sigma/artex_guard_audit_framing.yml`](../sigma/artex_guard_audit_framing.yml)で検知してください。
  サーバーポート `:8787` と記録プロキシ `127.0.0.1:8788`(`cmd/artex/main.go`)は、ネットワークシグネチャではなく
  ホストフォレンジック用(`ss`・`netstat`)です。

## 検証とテスト

Suricata 8 で検証しました。ロードテストはトラフィックが不要で、常に実行できます。

```sh
# 文法 + エンジンロードテスト (期待値: "Configuration provided was successfully loaded")
docker run --rm -v "$PWD/detections/suricata":/r -w /r jasonish/suricata:latest \
  suricata -T -S artex.rules -l /tmp --init-errors-fatal
```

`--init-errors-fatal` は、パースはできても初期化に失敗するルールもハードエラーにし、黙って破棄された
シグネチャがあればロードテストが通らないようにします。

ルールが実際に発火するかを確認するには、再現可能な回帰テストが [`../tests/suricata/`](../tests/suricata/)に
あります。まず同じロード確認を実行し、scapy で決定的なキャプチャを合成した後、その上で `suricata -r` を
実行してアラート数をアサートします。Docker さえあれば動作します。

```sh
detections/tests/suricata/run.sh
```

sid 1000001 がプローブごとにちょうど 1 回ずつ発火すること(35 フローのキャプチャで 35 回)、sid 1000002 が 300 秒間 30 件の
しきい値を超えること(Suricata 8.0.7 で **5** 回アラート、31~35 番目のフロー)、同じキャプチャを良性(benign)なブラウザ
User-Agent で実行するとアラートが **0** であることをアサートします。シグネチャが特異であることを確認するものです。
[`../tests/README.ko.md`](../tests/README.ko.md)を参照してください。代わりに自分のトラフィックで確認するには、ローカル
サーバーに対するループバック `curl -A 'artex-enrich/1.0'` をキャプチャしてアラートを読んでください。

```sh
suricata -r enrich.pcap -S artex.rules -l out && \
  grep -c '"signature_id":1000001' out/eve.json    # 存在: プローブごとに 1 回
```

## 貢献

検知への貢献を歓迎します。新しいルールは、すべての指標を観測可能な事実に基づかせ、限界をコメントに
明記し、`suricata -T` をクリーンに通過し、攻撃の手引きと読める内容を含まないようにしてください。
[`../../CONTRIBUTING.md`](../../CONTRIBUTING.md)と [`../sigma/`](../sigma/)の Sigma 層 /
[`../README.ko.md`](../README.ko.md)を参照してください。
