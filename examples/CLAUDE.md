# プロジェクトCLAUDE.mdの例

これはプロジェクトレベルのCLAUDE.mdファイルの例です。プロジェクトのルートに配置してください。

## プロジェクト概要

[あなたのプロジェクトの簡単な説明 - 何をするか、技術スタック]

## クリティカルなルール

### 1. コード構成

- 少数の大きなファイルより多数の小さなファイル
- 高い凝集度、低い結合度
- 通常200〜400行、ファイルあたり最大800行
- 型ではなく、機能/ドメインで整理する

### 2. コードスタイル

- コード、コメント、ドキュメントに絵文字を使用しない
- 常に不変性 - オブジェクトや配列を決して変更しない
- 本番コードにconsole.logを含めない
- try/catchによる適切なエラーハンドリング
- Zodまたは類似のライブラリによる入力検証

### 3. テスト

- TDD: 最初にテストを書く
- 最低80%のカバレッジ
- ユーティリティのための単体テスト
- APIのための統合テスト
- 重要なフローのためのE2Eテスト

### 4. セキュリティ

- ハードコードされたシークレットなし
- 機密データのための環境変数
- すべてのユーザー入力を検証する
- パラメータ化されたクエリのみ
- CSRF保護を有効にする

## ファイル構造

```
src/
|-- app/              # Next.js app router
|-- components/       # 再利用可能なUIコンポーネント
|-- hooks/            # カスタムReactフック
|-- lib/              # ユーティリティライブラリ
|-- types/            # TypeScript定義
```

## 主要なパターン

### APIレスポンス形式

```typescript
interface ApiResponse<T> {
  success: boolean
  data?: T
  error?: string
}
```

### エラーハンドリング

```typescript
try {
  const result = await operation()
  return { success: true, data: result }
} catch (error) {
  console.error('Operation failed:', error)
  return { success: false, error: 'User-friendly message' }
}
```

## 環境変数

```bash
# 必須
DATABASE_URL=
API_KEY=

# オプショナル
DEBUG=false
```

## 利用可能なコマンド

- `/tdd` - テスト駆動開発ワークフロー
- `/plan` - 実装計画の作成
- `/code-review` - コード品質のレビュー
- `/build-fix` - ビルドエラーの修正

## Gitワークフロー

- Conventional Commits: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`
- mainに直接コミットしない
- PRにはレビューが必要
- マージ前にすべてのテストがパスする必要がある
