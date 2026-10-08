# `/btw` 検証記録

日本語 · [中文](VALIDATION.zh.md)

日付: 2026-09-10。ブランチ: `codex/btw-side-question`。基準コミット(baseline): `8dae851b9b622f2ff2631f332fde9719d0b16fba`。

> この文書は、原本の中国語文書(`VALIDATION.zh.md`)を日本語に翻訳したものです。アップストリーム(upstream)リポジトリの変更と照合しやすいように、原本はそのまま保存しています。

独立した PostgreSQL テスト DB とデータディレクトリを使用しました。実際のモデル認証情報は独立したテスト環境にのみ注入し、コードやこの記録には書いておらず、製品のデフォルトモデルも変更していません。Go 1.26.3、norma v0.3.6、Next.js 16.2.9。

実際のモデル会話、返却オブジェクト、エンジニアリング上のアサーション、Qwen 原本の審査テキストは [validation-2026-09-10.json](validation-2026-09-10.json) に保存しており、その中に API 認証情報は含まれていません。

## エンジニアリング検査

以下の項目はすべて通過しており、括弧内は根拠となるテストです。

- 構造化メッセージとツール引数のディープコピー: 通過(根拠 `TestCheckpointDeepCopyAndBoundaries`)。
- 要約・圧縮リクエストが上書きしないこと、完結した応答と終了状態の発行、生成途中の半端な応答の除外: 通過(根拠 `TestCheckpointDeepCopyAndBoundaries`, `TestSnapshotExcludesPartialStreamAndSelectsPoolMember`)。
- 実際のモデルプール構成員の識別: 通過(根拠 `TestSnapshotExcludesPartialStreamAndSelectsPoolMember`)。
- ツールのペアリング、20 組の再生、予算の削減と超過エラー: 通過(根拠 `TestBuildRequestCompactionToolPairingAndBudget`)。
- メインとサイドクエスチョンの並列実行、双方向のキャンセル分離: 通過(根拠: ブロッキング方式の Provider、`TestMainSideConcurrencyAndIndependentCancellation`)。
- ツール実行なし、ストリーミング・非ストリーミング、失敗時点の既存の使用量: 通過(根拠 `TestServiceNoToolsAndUsageOnFailure`)。
- 実際の norma ChatAgent とローカルの Read ツール、メイン transcript・アクティビティの分離: 通過(根拠 `TestSideActualChatCheckpointToolResultAndTranscriptIsolation` のストリーミング・非ストリーミングのサブケース)。
- 永続化、ページ分割、冪等性、再起動後の部分応答の保持: 通過(根拠 `TestSideHistoryIdempotencyPagingAndRecovery`)。
- クリアと遅れた書き込みの競合、親リソースの削除、バージョン比較: 通過(根拠 `TestSideClearLateWritersAndDeletedParent`)。
- MainAgent・Worker のアーカイブと復元(v1・v2・v3): 通過(根拠 `TestSideTaskArchiveVersions`)。
- 3 種類の親インターフェース、認証、リソースの帰属、Worker の論理削除: 通過(根拠 `TestSideHTTPGlobalLimitTaskWorkerAndDeletion`, `TestSideCheckpointPersistsBeforeAdmissionAndRestart`)。
- ビジー状態のメインセッションでもサイドクエスチョンが可能、独立した SSE の再接続・切断、キャンセル、クリア: 通過(根拠 `TestSideHTTPBusyIsolationClearAndReconnect`)。
- 親セッションあたり 1 件・全体で 4 件の同時実行: 通過(根拠: 二つの `TestSideHTTP…` ケース)。
- 送信前のスナップショット保存、再起動後の続きの質問、古いセッションがスナップショットを偽造できないこと: 通過(根拠 `TestSideCheckpointPersistsBeforeAdmissionAndRestart`)。
- キャッシュにある設定が削除されたりモデルが変更されたりした場合は、継続を拒否: 通過(根拠 `TestSideRejectsDeletedOrChangedCachedProfile`)。
- アーカイブ前にキャンセルし、最終応答と使用量が保存されるのを待つ: 通過(根拠 `TestSideTaskDrainPersistsBeforeArchive`)。
- ストリーミングの消費側が早期にキャンセルしても、使用量を 1 回だけ記録してサイドクエスチョンに帰属: 通過(根拠 `TestSideUsageRecordedOnceOnConsumerCancellation`)。
- 再起動によって自動復元された Worker・deadline の実行コンテキストが、引き続き新しいスナップショットを発行: 通過(根拠 `TestSideRestoredWorkerRuntimePublishesNewCheckpoint`)。
- 関連パッケージの race 検査: 通過(根拠: 下記コマンド)。
- TypeScript とプロダクションビルド: 通過(根拠 `npx tsc --noEmit`, `npm run build`)。
- 新規追加したフロントエンドモジュールの Biome 検査: 通過(根拠 `biome check`、新規モジュール 3 つ)。

破棄してもよいデータベースに `ARTEX_PG_DSN` を設定すれば、自動テストを再現できます(本番 DB を指さないでください):

```sh
go test -race ./agent ./db ./server ./sidequestion ./llmrec ./llmpool \
  -run 'Test(Side|Checkpoint|Snapshot|BuildRequest|Service|MainSide|CaptureRun|TaskArchive|CompleteForwards|StopIntent|CancelIntent)' -count=1
cd web
npx tsc --noEmit
npx biome check src/lib/side-questions.ts src/hooks/use-side-questions.ts src/components/side-question-workspace.tsx
npm run build
```

Go の全体回帰テストはすべて通過(green)しているわけではありません。`server` パッケージの既存テスト 2 つが一時ディレクトリのクリーンアップ段階で失敗しており、どちらも `TempDir RemoveAll … directory not empty` を報告します:

- `TestInheritedActivityDetailAndRelationDeletion`
- `TestTaskMetadataPatchReturnsRenameAndPin`

上記の未修正の基準コミットからソースを取得し、同じ隔離環境で `server` パッケージを再度実行しても、この 2 つのクリーンアップ失敗はまったく同じように再現されます。基準コミットでの実行では、`TestCoreTaskLifecyclePG` の対象ノード数のアサーション失敗も別途現れましたが、最終的な修正版の `server` 回帰ではそのアサーション失敗はありませんでした。他のパッケージは通過しており、今回のサイドクエスチョン関連のケースと race 検査も通過しました。基準コミットの問題を今回の検収の通過として扱ってはおらず、隠れた問題を覆い隠すために既存のアサーションを変更することもしていません。

Next.js のビルドは、複数の lockfile・workspace ルートの推論に関する既存の警告を出力しますが、ビルドは完了し、すべてのページが正常に生成されます。

## ブラウザー検査

Codex の内蔵ブラウザーで、独立したローカルの Go サービスと Next.js 開発サーバーに接続しました。デスクトップと 390 × 844 の狭い画面で、次の手動・自動操作を行い、スクリーンショットとブラウザーログを確認しました:

- 通常のチャットが実行されている間に `/btw` を入力すると、本文の内容とサイドクエスチョンが同時に表示され、デスクトップのサイドバーも正常です。
- 続けて追加の質問をしました。サイドクエスチョンを停止しても、すでに生成された部分は残り、本文のフローは続行します。
- パネルを閉じてもリクエストは続行し、再度開くと完結した応答を復元します。ページを再読み込みしたあと、内容なしで `/btw` だけを入力すると履歴を復元します。
- 狭い画面の Drawer で、入力、ボタン、履歴、閉じる操作が正常で、横方向のはみ出しはありません。
- クリアは確認ポップアップを表示し、クリア後は履歴が消えますが、メイン transcript とスナップショットはそのまま残ります。
- タスクの MainAgent と 2 つの Worker にそれぞれ質問して切り替えたところ、エージェントのタブと履歴が互いに混ざることはありませんでした。
- ブロッキング方式のローカルモデルフィクスチャで Worker を実行し続けた状態で、Worker のメイン入力欄から `/btw` を送信しました。サイドクエスチョンを停止したあとも、Worker はリアルタイムの実行状態と自身の一時停止ボタンをそのまま表示し、サイドクエスチョンは部分応答を保存しました。
- ブラウザーのエラー・警告ログは空です。

制御可能なフィクスチャは、同時実行のタイミングを精密に検証するために使ったもので、実際のモデルの出力速度には依存しません。デバッグ中の 2 回の Worker 実行検査では、有効な同時実行区間が作られませんでしたが(タスクがすでに終了していた、または応答が先に完了していた)、フィクスチャを修正して再実行し、通過しました。この初期の操作は、有効な通過としては数えません。

## 実際のモデル会話

まず `grok-4.6` を探索しました。OpenAI 互換インターフェースは `http://127.0.0.1:12580/tingly/openai` です。探索の結果、HTTP 200 とともにモデル名 `grok-4.6` と `READY` を返し、2.82 秒かかりました。第 1 優先が利用可能だったため、Tingly の `glm` や Zhipu の `glm-5.3` の予備チェーンは有効にしておらず、この 2 つの予備サービスは今回検証していません。

- メインセッションの実行中に資産・目標・マーカーを質問: `redhaze.top`、トップページの読み取りと目標の要約、`BTW-REAL-0910` を返し、サイドクエスチョンが完了(16.97 秒)。
- メインセッションがトップページの読み取りを終えたあとにツールの根拠を質問: WebFetch 200、curl の 301 → 302 → 200 のリダイレクト、ページタイトルを正確に引用(7.24 秒)。
- サイドクエスチョンが Bash でテストファイルを作るよう要求: 実行を拒否し、対象ファイルは作成されませんでした(7.74 秒)。
- 完了後のサイドクエスチョンがメインコンテキストを変更しないこと: メイン transcript の SHA-256 とメインのアクティビティ記録がそのまま一致し、サイドクエスチョンのツール実行回数は 0。
- Go サービスを実際に停止・再起動したあとに続けて質問: 以前のサイドクエスチョン履歴 3 件を保持しており、永続化したスナップショットからすぐに資産・マーカー・タイトルを答え、メインエージェントを再度実行しませんでした。
- 新しいセッションで Grok の非ストリーミング設定を使用: 資産と `ATOMIC-0910` を正確に答え、使用量を返却・保存(input 11734, output 138, cache_read 11520)。

資産ケースのメインセッションは、WebFetch と Bash/curl で公開されているトップページを読み取り、ランディングページは `https://id.redhaze.top/home`、ページが返したタイトルは "红幕科技 RedHaze Group · 全球综合集团门户" でした(モデルが返した原文なのでそのまま引用)。Bash は応答をローカルのテストファイルに一時的に保存しただけで、リモートへの書き込みは実行していません。この事実は、「サイドクエスチョンがツールを実行しなかった」という点とは別に確認しました。

メイン transcript の検証値: `e7e61f135a4a120954b539f357e8c4205d7d5cd7460dcaf3dc0fd066463e1d00`。

**使用量の制限:** Tingly 経由の Grok のストリーミング応答は usage を返しませんでした。別途 `stream_options.include_usage=true` を直接送って検証したところ、HTTP 200、データフレーム 12 個、usage フレーム 0 個でした。したがって、ストリーミングテストでの 0 は、エンドポイントが使用量を提供しないことを意味し、課金がなかったという意味に解釈してはいけません。非ストリーミングの使用量と、フィクスチャでの失敗・キャンセル時の使用量は、いずれも正しく保存されました。

## Qwen による審査

審査モデルは `qwen-flash`、OpenAI 互換インターフェースは `https://dashscope.aliyuncs.com/compatible-mode/v1`、HTTP 200 です。前述の 3 つの実際のサイドクエスチョン会話、メインセッションのツールの根拠、エンジニアリング上のアサーションを提供し、`verdict: accept` と `concerns: []` を返しました。応答が資産・マーカー・ページ読み取りの証拠と一致しており、サイドクエスチョンのツール拒否が制約に適合していると判断しました。審査の使用量: prompt 6625、completion 312、total 6937。

今回の Qwen による審査の範囲には、後から追加したサービスの再起動と非ストリーミングのテストは含まれていません。Qwen の「書き込みなし」という一般化は広すぎます。メインセッションの curl は実際にローカルの応答用一時ファイルを作成しており、これは前述のとおり明確に記録しました。同時実行性、ツール実行 0 回、transcript の分離はエンジニアリング上のアサーションで判断し、モデルによる審査は応答品質の評価を補助するにとどまります。
