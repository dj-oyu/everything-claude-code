# コーディングスタイル

## 不変性（クリティカル）

常に新しいオブジェクトを作成し、決してミューテート（変更）しないこと：

```javascript
// 誤り: ミューテーション
function updateUser(user, name) {
  user.name = name  // ミューテーション!
  return user
}

// 正しい: 不変性
function updateUser(user, name) {
  return {
    ...user,
    name
  }
}
```

## ファイル構成

少数の大きなファイルより多数の小さなファイル：
- 高い凝集度、低い結合度
- 通常200〜400行、最大800行
- 大きなコンポーネントからユーティリティを抽出する
- 型ではなく、機能/ドメインで整理する

## エラー処理

常にエラーを包括的に処理すること：

```typescript
try {
  const result = await riskyOperation()
  return result
} catch (error) {
  console.error('Operation failed:', error)
  throw new Error('Detailed user-friendly message')
}
```

## 入力検証

常にユーザー入力を検証すること：

```typescript
import { z } from 'zod'

const schema = z.object({
  email: z.string().email(),
  age: z.number().int().min(0).max(150)
})

const validated = schema.parse(input)
```

## コード品質チェックリスト

作業完了とマークする前に：
- [ ] コードは読みやすく、適切に命名されている
- [ ] 関数は小さい（<50行）
- [ ] ファイルは焦点が絞られている（<800行）
- [ ] 深いネストがない（>4レベル）
- [ ] 適切なエラー処理
- [ ] `console.log`文がない
- [ ] ハードコードされた値がない
- [ ] ミューテーションがない（不変パターンが使用されている）
