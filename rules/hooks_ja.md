# フックシステム

## フックの種類

- **PreToolUse**: ツール実行前（検証、パラメータ変更）
- **PostToolUse**: ツール実行後（自動フォーマット、チェック）
- **Stop**: セッション終了時（最終検証）

## 現在のフック (`~/.claude/settings.json` 内)

### PreToolUse
- **tmuxリマインダー**: 長時間実行コマンド（npm, pnpm, yarn, cargoなど）にtmuxを提案
- **git pushレビュー**: プッシュ前にZedを開いてレビュー
- **docブロッカー**: 不要な.md/.txtファイルの作成をブロック

### PostToolUse
- **PR作成**: PRのURLとGitHub Actionsのステータスをログに記録
- **Prettier**: 編集後にJS/TSファイルを自動フォーマット
- **TypeScriptチェック**: .ts/.tsxファイル編集後にtscを実行
- **console.log警告**: 編集されたファイル内のconsole.logについて警告

### Stop
- **console.log監査**: セッション終了前に、変更されたすべてのファイルのconsole.logをチェック

## 自動承認パーミッション

注意して使用すること：
- 信頼できる、明確に定義された計画に対して有効にする
- 探索的な作業では無効にする
- `dangerously-skip-permissions`フラグは決して使用しない
- 代わりに`~/.claude.json`で`allowedTools`を設定する

## TodoWriteのベストプラクティス

TodoWriteツールを使用して：
- 複数ステップのタスクの進捗を追跡する
- 指示の理解度を確認する
- リアルタイムの操縦を可能にする
- 詳細な実装ステップを示す

Todoリストが明らかにすること：
- 順序が違うステップ
- 不足している項目
- 余分な不要な項目
- 間違った粒度
- 誤解された要件
