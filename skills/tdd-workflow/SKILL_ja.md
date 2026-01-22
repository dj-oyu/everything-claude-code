---
name: tdd-workflow
description: 新機能の作成、バグ修正、コードのリファクタリング時にこのスキルを使用します。単体、統合、E2Eテストを含む80%以上のカバレッジでテスト駆動開発を強制します。
---

# テスト駆動開発ワークフロー

このスキルは、すべてのコード開発が包括的なテストカバレッジを持つTDD原則に従うことを保証します。

## アクティベートするタイミング

- 新機能や機能性の作成
- バグや問題の修正
- 既存コードのリファクタリング
- APIエンドポイントの追加
- 新しいコンポーネントの作成

## コア原則

### 1. テストをコードの前に
常に最初にテストを書き、次にテストをパスさせるためのコードを実装します。

### 2. カバレッジ要件
- 最低80%のカバレッジ（単体 + 統合 + E2E）
- すべてのエッジケースをカバー
- エラーシナリオをテスト
- 境界条件を検証

### 3. テストの種類

#### 単体テスト
- 個々の関数とユーティリティ
- コンポーネントロジック
- 純粋関数
- ヘルパーとユーティリティ

#### 統合テスト
- APIエンドポイント
- データベース操作
- サービスインタラクション
- 外部API呼び出し

#### E2Eテスト (Playwright)
- 重要なユーザーフロー
- 完全なワークフロー
- ブラウザ自動化
- UIインタラクション

## TDDワークフローの手順

### ステップ1: ユーザージャーニーを作成する
```
[役割]として、[利益]のために[アクション]をしたい

例:
ユーザーとして、意味的に市場を検索したい、
そうすれば、正確なキーワードがなくても関連する市場を見つけることができる。
```

### ステップ2: テストケースを生成する
各ユーザージャーニーについて、包括的なテストケースを作成します:

```typescript
describe('Semantic Search', () => {
  it('returns relevant markets for query', async () => {
    // テスト実装
  })

  it('handles empty query gracefully', async () => {
    // エッジケースのテスト
  })

  it('falls back to substring search when Redis unavailable', async () => {
    // フォールバック挙動のテスト
  })

  it('sorts results by similarity score', async () => {
    // ソートロジックのテスト
  })
})
```

### ステップ3: テストを実行する（失敗するはず）
```bash
npm test
# テストは失敗するはずです - まだ実装していません
```

### ステップ4: コードを実装する
テストをパスさせるための最小限のコードを書きます:

```typescript
// テストに導かれた実装
export async function searchMarkets(query: string) {
  // ここに実装
}
```

### ステップ5: 再度テストを実行する
```bash
npm test
# テストは अब成功するはずです
```

### ステップ6: リファクタリング
テストをグリーンのままコード品質を向上させます:
- 重複を削除
- 命名を改善
- パフォーマンスを最適化
- 可読性を向上

### ステップ7: カバレッジを検証する
```bash
npm run test:coverage
# 80%以上のカバレッジが達成されたことを確認
```

## テストパターン

### 単体テストパターン (Jest/Vitest)
```typescript
import { render, screen, fireEvent } from '@testing-library/react'
import { Button } from './Button'

describe('Button Component', () => {
  it('renders with correct text', () => {
    render(<Button>Click me</Button>)
    expect(screen.getByText('Click me')).toBeInTheDocument()
  })

  it('calls onClick when clicked', () => {
    const handleClick = jest.fn()
    render(<Button onClick={handleClick}>Click</Button>)

    fireEvent.click(screen.getByRole('button'))

    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  it('is disabled when disabled prop is true', () => {
    render(<Button disabled>Click</Button>)
    expect(screen.getByRole('button')).toBeDisabled()
  })
})
```

### API統合テストパターン
```typescript
import { NextRequest } from 'next/server'
import { GET } from './route'

describe('GET /api/markets', () => {
  it('returns markets successfully', async () => {
    const request = new NextRequest('http://localhost/api/markets')
    const response = await GET(request)
    const data = await response.json()

    expect(response.status).toBe(200)
    expect(data.success).toBe(true)
    expect(Array.isArray(data.data)).toBe(true)
  })

  it('validates query parameters', async () => {
    const request = new NextRequest('http://localhost/api/markets?limit=invalid')
    const response = await GET(request)

    expect(response.status).toBe(400)
  })

  it('handles database errors gracefully', async () => {
    // データベースの失敗をモックする
    const request = new NextRequest('http://localhost/api/markets')
    // エラーハンドリングをテストする
  })
})
```

### E2Eテストパターン (Playwright)
```typescript
import { test, expect } from '@playwright/test'

test('user can search and filter markets', async ({ page }) => {
  // マーケットページに移動
  await page.goto('/')
  await page.click('a[href="/markets"]')

  // ページが読み込まれたことを確認
  await expect(page.locator('h1')).toContainText('Markets')

  // マーケットを検索
  await page.fill('input[placeholder="Search markets"]', 'election')

  // デバウンスと結果を待つ
  await page.waitForTimeout(600)

  // 検索結果が表示されたことを確認
  const results = page.locator('[data-testid="market-card"]')
  await expect(results).toHaveCount(5, { timeout: 5000 })

  // 結果に検索語が含まれていることを確認
  const firstResult = results.first()
  await expect(firstResult).toContainText('election', { ignoreCase: true })

  // ステータスでフィルタリング
  await page.click('button:has-text("Active")')

  // フィルタリングされた結果を確認
  await expect(results).toHaveCount(3)
})

test('user can create a new market', async ({ page }) => {
  // 最初にログイン
  await page.goto('/creator-dashboard')

  // マーケット作成フォームを入力
  await page.fill('input[name="name"]', 'Test Market')
  await page.fill('textarea[name="description"]', 'Test description')
  await page.fill('input[name="endDate"]', '2025-12-31')

  // フォームを送信
  await page.click('button[type="submit"]')

  // 成功メッセージを確認
  await expect(page.locator('text=Market created successfully')).toBeVisible()

  // マーケットページへのリダイレクトを確認
  await expect(page).toHaveURL(/\/markets\/test-market/)
})
```

## テストファイルの構成

```
src/
├── components/
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx          # 単体テスト
│   │   └── Button.stories.tsx       # Storybook
│   └── MarketCard/
│       ├── MarketCard.tsx
│       └── MarketCard.test.tsx
├── app/
│   └── api/
│       └── markets/
│           ├── route.ts
│           └── route.test.ts         # 統合テスト
└── e2e/
    ├── markets.spec.ts               # E2Eテスト
    ├── trading.spec.ts
    └── auth.spec.ts
```

## 外部サービスのモック

### Supabaseモック
```typescript
jest.mock('@/lib/supabase', () => ({
  supabase: {
    from: jest.fn(() => ({
      select: jest.fn(() => ({
        eq: jest.fn(() => Promise.resolve({
          data: [{ id: 1, name: 'Test Market' }],
          error: null
        }))
      }))
    }))
  }
}))
```

### Redisモック
```typescript
jest.mock('@/lib/redis', () => ({
  searchMarketsByVector: jest.fn(() => Promise.resolve([
    { slug: 'test-market', similarity_score: 0.95 }
  ])),
  checkRedisHealth: jest.fn(() => Promise.resolve({ connected: true }))
}))
```

### OpenAIモック
```typescript
jest.mock('@/lib/openai', () => ({
  generateEmbedding: jest.fn(() => Promise.resolve(
    new Array(1536).fill(0.1) // 1536次元の埋め込みをモック
  ))
}))
```

## テストカバレッジの検証

### カバレッジレポートの実行
```bash
npm run test:coverage
```

### カバレッジのしきい値
```json
{
  "jest": {
    "coverageThresholds": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    }
  }
}
```

## 避けるべき一般的なテストの間違い

### ❌ 誤り: 実装の詳細をテストする
```typescript
// 内部状態をテストしない
expect(component.state.count).toBe(5)
```

### ✅ 正しい: ユーザーに見える振る舞いをテストする
```typescript
// ユーザーが見るものをテストする
expect(screen.getByText('Count: 5')).toBeInTheDocument()
```

### ❌ 誤り: 壊れやすいセレクタ
```typescript
// 簡単に壊れる
await page.click('.css-class-xyz')
```

### ✅ 正しい: セマンティックなセレクタ
```typescript
// 変更に強い
await page.click('button:has-text("Submit")')
await page.click('[data-testid="submit-button"]')
```

### ❌ 誤り: テストの分離がない
```typescript
// テストが互いに依存している
test('creates user', () => { /* ... */ })
test('updates same user', () => { /* 前のテストに依存 */ })
```

### ✅ 正しい: 独立したテスト
```typescript
// 各テストが自身のデータをセットアップする
test('creates user', () => {
  const user = createTestUser()
  // テストロジック
})

test('updates user', () => {
  const user = createTestUser()
  // 更新ロジック
})
```

## 継続的テスト

### 開発中のウォッチモード
```bash
npm test -- --watch
# ファイル変更時にテストが自動実行される
```

### プリコミットフック
```bash
# すべてのコミットの前に実行される
npm test && npm run lint
```

### CI/CD統合
```yaml
# GitHub Actions
- name: Run Tests
  run: npm test -- --coverage
- name: Upload Coverage
  uses: codecov/codecov-action@v3
```

## ベストプラクティス

1. **最初にテストを書く** - 常にTDD
2. **テストごとに1つのアサート** - 単一の振る舞いに焦点を当てる
3. **説明的なテスト名** - テスト内容を説明する
4. **準備・実行・検証** - 明確なテスト構造
5. **外部依存関係をモックする** - 単体テストを分離する
6. **エッジケースをテストする** - Null, undefined, empty, large
7. **エラーパスをテストする** - ハッピーパスだけでなく
8. **テストを高速に保つ** - 単体テストはそれぞれ < 50ms
9. **テスト後にクリーンアップする** - 副作用なし
10. **カバレッジレポートを確認する** - ギャップを特定する

## 成功の指標

- 80%以上のコードカバレッジ達成
- すべてのテストが成功（グリーン）
- スキップまたは無効化されたテストがない
- 高速なテスト実行（単体テスト < 30秒）
- E2Eテストが重要なユーザーフローをカバー
- テストが本番前にバグをキャッチ

---

**忘れないで**: テストは任意ではありません。それらは、自信を持ったリファクタリング、迅速な開発、そして本番環境の信頼性を可能にするセーフティネットです。
