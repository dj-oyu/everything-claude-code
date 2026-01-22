---
name: coding-standards
description: TypeScript, JavaScript, React, Node.js開発のための普遍的なコーディング標準、ベストプラクティス、およびパターン。
---

# コーディング標準とベストプラクティス

すべてのプロジェクトに適用可能な普遍的なコーディング標準。

## コード品質の原則

### 1. 可読性第一
- コードは書かれるよりも読まれることの方が多い
- 明確な変数名と関数名
- コメントよりも自己文書化されたコードを優先
- 一貫したフォーマット

### 2. KISS (Keep It Simple, Stupid)
- 機能する最もシンプルな解決策
- 過剰なエンジニアリングを避ける
- 早すぎる最適化をしない
- 巧妙なコードよりも理解しやすいコード

### 3. DRY (Don't Repeat Yourself)
- 共通のロジックを関数に抽出する
- 再利用可能なコンポーネントを作成する
- モジュール間でユーティリティを共有する
- コピー＆ペーストプログラミングを避ける

### 4. YAGNI (You Aren't Gonna Need It)
- 必要になる前に機能を構築しない
- 推測に基づく一般化を避ける
- 必要な場合にのみ複雑さを追加する
- シンプルに始め、必要なときにリファクタリングする

## TypeScript/JavaScript標準

### 変数名

```typescript
// ✅ 良い: 説明的な名前
const marketSearchQuery = 'election'
const isUserAuthenticated = true
const totalRevenue = 1000

// ❌ 悪い: 不明瞭な名前
const q = 'election'
const flag = true
const x = 1000
```

### 関数名

```typescript
// ✅ 良い: 動詞-名詞パターン
async function fetchMarketData(marketId: string) { }
function calculateSimilarity(a: number[], b: number[]) { }
function isValidEmail(email: string): boolean { }

// ❌ 悪い: 不明瞭または名詞のみ
async function market(id: string) { }
function similarity(a, b) { }
function email(e) { }
```

### 不変性パターン (クリティカル)

```typescript
// ✅ 常にスプレッド演算子を使用する
const updatedUser = {
  ...user,
  name: 'New Name'
}

const updatedArray = [...items, newItem]

// ❌ 直接ミューテートしない
user.name = 'New Name'  // 悪い
items.push(newItem)     // 悪い
```

### エラーハンドリング

```typescript
// ✅ 良い: 包括的なエラーハンドリング
async function fetchData(url: string) {
  try {
    const response = await fetch(url)

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`)
    }

    return await response.json()
  } catch (error) {
    console.error('Fetch failed:', error)
    throw new Error('Failed to fetch data')
  }
}

// ❌ 悪い: エラーハンドリングなし
async function fetchData(url) {
  const response = await fetch(url)
  return response.json()
}
```

### Async/Awaitのベストプラクティス

```typescript
// ✅ 良い: 可能な場合は並列実行
const [users, markets, stats] = await Promise.all([
  fetchUsers(),
  fetchMarkets(),
  fetchStats()
])

// ❌ 悪い: 不要な場合の逐次実行
const users = await fetchUsers()
const markets = await fetchMarkets()
const stats = await fetchStats()
```

### 型安全性

```typescript
// ✅ 良い: 適切な型
interface Market {
  id: string
  name: string
  status: 'active' | 'resolved' | 'closed'
  created_at: Date
}

function getMarket(id: string): Promise<Market> {
  // 実装
}

// ❌ 悪い: 'any'の使用
function getMarket(id: any): Promise<any> {
  // 実装
}
```

## Reactのベストプラクティス

### コンポーネント構造

```typescript
// ✅ 良い: 型付きの関数コンポーネント
interface ButtonProps {
  children: React.ReactNode
  onClick: () => void
  disabled?: boolean
  variant?: 'primary' | 'secondary'
}

export function Button({
  children,
  onClick,
  disabled = false,
  variant = 'primary'
}: ButtonProps) {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className={`btn btn-${variant}`}
    >
      {children}
    </button>
  )
}

// ❌ 悪い: 型がなく、不明瞭な構造
export function Button(props) {
  return <button onClick={props.onClick}>{props.children}</button>
}
```

### カスタムフック

```typescript
// ✅ 良い: 再利用可能なカスタムフック
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}

// 使用法
const debouncedQuery = useDebounce(searchQuery, 500)
```

### 状態管理

```typescript
// ✅ 良い: 適切な状態更新
const [count, setCount] = useState(0)

// 以前の状態に基づく状態のための関数更新
setCount(prev => prev + 1)

// ❌ 悪い: 直接的な状態参照
setCount(count + 1)  // 非同期シナリオでは古くなる可能性がある
```

### 条件付きレンダリング

```typescript
// ✅ 良い: 明確な条件付きレンダリング
{isLoading && <Spinner />}
{error && <ErrorMessage error={error} />}
{data && <DataDisplay data={data} />}

// ❌ 悪い: 三項演算子の地獄
{isLoading ? <Spinner /> : error ? <ErrorMessage error={error} /> : data ? <DataDisplay data={data} /> : null}
```

## API設計標準

### REST API規約

```
GET    /api/markets              # 全マーケットを一覧表示
GET    /api/markets/:id          # 特定のマーケットを取得
POST   /api/markets              # 新しいマーケットを作成
PUT    /api/markets/:id          # マーケットを更新 (全体)
PATCH  /api/markets/:id          # マーケットを更新 (部分的)
DELETE /api/markets/:id          # マーケットを削除

# フィルタリングのためのクエリパラメータ
GET /api/markets?status=active&limit=10&offset=0
```

### レスポンス形式

```typescript
// ✅ 良い: 一貫したレスポンス構造
interface ApiResponse<T> {
  success: boolean
  data?: T
  error?: string
  meta?: {
    total: number
    page: number
    limit: number
  }
}

// 成功レスポンス
return NextResponse.json({
  success: true,
  data: markets,
  meta: { total: 100, page: 1, limit: 10 }
})

// エラーレスポンス
return NextResponse.json({
  success: false,
  error: 'Invalid request'
}, { status: 400 })
```

### 入力検証

```typescript
import { z } from 'zod'

// ✅ 良い: スキーマ検証
const CreateMarketSchema = z.object({
  name: z.string().min(1).max(200),
  description: z.string().min(1).max(2000),
  endDate: z.string().datetime(),
  categories: z.array(z.string()).min(1)
})

export async function POST(request: Request) {
  const body = await request.json()

  try {
    const validated = CreateMarketSchema.parse(body)
    // 検証済みデータで続行
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json({
        success: false,
        error: 'Validation failed',
        details: error.errors
      }, { status: 400 })
    }
  }
}
```

## ファイル構成

### プロジェクト構造

```
src/
├── app/                    # Next.js App Router
│   ├── api/               # APIルート
│   ├── markets/           # マーケットページ
│   └── (auth)/           # 認証ページ (ルートグループ)
├── components/            # Reactコンポーネント
│   ├── ui/               # 汎用UIコンポーネント
│   ├── forms/            # フォームコンポーネント
│   └── layouts/          # レイアウトコンポーネント
├── hooks/                # カスタムReactフック
├── lib/                  # ユーティリティと設定
│   ├── api/             # APIクライアント
│   ├── utils/           # ヘルパー関数
│   └── constants/       # 定数
├── types/                # TypeScript型
└── styles/              # グローバルスタイル
```

### ファイル名

```
components/Button.tsx          # コンポーネントはPascalCase
hooks/useAuth.ts              # 'use'プレフィックス付きのcamelCase
lib/formatDate.ts             # ユーティリティはcamelCase
types/market.types.ts         # .typesサフィックス付きのcamelCase
```

## コメントとドキュメンテーション

### コメントするタイミング

```typescript
// ✅ 良い: なぜかを説明し、何をかを説明しない
// 障害発生時にAPIを圧倒しないように、指数バックオフを使用する
const delay = Math.min(1000 * Math.pow(2, retryCount), 30000)

// 大規模な配列のパフォーマンスのために意図的にミューテーションを使用している
items.push(newItem)

// ❌ 悪い: 当たり前のことを述べる
// カウンターを1増やす
count++

// 名前をユーザーの名前に設定する
name = user.name
```

### 公開APIのためのJSDoc

```typescript
/**
 * セマンティックな類似性を使用してマーケットを検索します。
 *
 * @param query - 自然言語の検索クエリ
 * @param limit - 最大結果数 (デフォルト: 10)
 * @returns 類似性スコアでソートされたマーケットの配列
 * @throws {Error} OpenAI APIが失敗した場合、またはRedisが利用できない場合
 *
 * @example
 * ```typescript
 * const results = await searchMarkets('election', 5)
 * console.log(results[0].name) // "Trump vs Biden"
 * ```
 */
export async function searchMarkets(
  query: string,
  limit: number = 10
): Promise<Market[]> {
  // 実装
}
```

## パフォーマンスのベストプラクティス

### メモ化

```typescript
import { useMemo, useCallback } from 'react'

// ✅ 良い: 高価な計算をメモ化する
const sortedMarkets = useMemo(() => {
  return markets.sort((a, b) => b.volume - a.volume)
}, [markets])

// ✅ 良い: コールバックをメモ化する
const handleSearch = useCallback((query: string) => {
  setSearchQuery(query)
}, [])
```

### 遅延読み込み

```typescript
import { lazy, Suspense } from 'react'

// ✅ 良い: 重いコンポーネントを遅延読み込みする
const HeavyChart = lazy(() => import('./HeavyChart'))

export function Dashboard() {
  return (
    <Suspense fallback={<Spinner />}>
      <HeavyChart />
    </Suspense>
  )
}
```

### データベースクエリ

```typescript
// ✅ 良い: 必要なカラムのみを選択する
const { data } = await supabase
  .from('markets')
  .select('id, name, status')
  .limit(10)

// ❌ 悪い: すべてを選択する
const { data } = await supabase
  .from('markets')
  .select('*')
```

## テスト標準

### テスト構造 (AAAパターン)

```typescript
test('calculates similarity correctly', () => {
  // Arrange (準備)
  const vector1 = [1, 0, 0]
  const vector2 = [0, 1, 0]

  // Act (実行)
  const similarity = calculateCosineSimilarity(vector1, vector2)

  // Assert (検証)
  expect(similarity).toBe(0)
})
```

### テスト名

```typescript
// ✅ 良い: 説明的なテスト名
test('returns empty array when no markets match query', () => { })
test('throws error when OpenAI API key is missing', () => { })
test('falls back to substring search when Redis unavailable', () => { })

// ❌ 悪い: 曖昧なテスト名
test('works', () => { })
test('test search', () => { })
```

## コードの臭いの検出

これらのアンチパターンに注意してください:

### 1. 長い関数
```typescript
// ❌ 悪い: 50行を超える関数
function processMarketData() {
  // 100行のコード
}

// ✅ 良い: 小さな関数に分割する
function processMarketData() {
  const validated = validateData()
  const transformed = transformData(validated)
  return saveData(transformed)
}
```

### 2. 深いネスト
```typescript
// ❌ 悪い: 5レベル以上のネスト
if (user) {
  if (user.isAdmin) {
    if (market) {
      if (market.isActive) {
        if (hasPermission) {
          // 何かをする
        }
      }
    }
  }
}

// ✅ 良い: アーリーリターン
if (!user) return
if (!user.isAdmin) return
if (!market) return
if (!market.isActive) return
if (!hasPermission) return

// 何かをする
```

### 3. マジックナンバー
```typescript
// ❌ 悪い: 説明のない数字
if (retryCount > 3) { }
setTimeout(callback, 500)

// ✅ 良い: 名前付き定数
const MAX_RETRIES = 3
const DEBOUNCE_DELAY_MS = 500

if (retryCount > MAX_RETRIES) { }
setTimeout(callback, DEBOUNCE_DELAY_MS)
```

**忘れないで**: コード品質は交渉の余地がありません。明確で保守しやすいコードは、迅速な開発と自信を持ったリファクタリングを可能にします。
