---
description: テスト駆動開発ワークフローを強制します。最初にインターフェースをスキャフォールドし、テストを生成してから、パスするための最小限のコードを実装します。80%以上のカバレッジを保証します。
---

# TDDコマンド

このコマンドは **tdd-guide** エージェントを呼び出し、テスト駆動開発の方法論を強制します。

## このコマンドの機能

1. **インターフェースのスキャフォールド** - 最初に型/インターフェースを定義します
2. **最初にテストを生成** - 失敗するテストを書きます (レッド)
3. **最小限のコードを実装** - パスするのに十分なコードだけを書きます (グリーン)
4. **リファクタリング** - テストをグリーンのままコードを改善します (リファクター)
5. **カバレッジの検証** - 80%以上のテストカバレッジを保証します

## 使用する場面

`/tdd` は以下の場合に使用します:
- 新機能の実装
- 新しい関数/コンポーネントの追加
- バグの修正（最初にバグを再現するテストを書く）
- 既存コードのリファクタリング
- クリティカルなビジネスロジックの構築

## 仕組み

tdd-guideエージェントは以下を実行します:

1. **入力/出力のインターフェースを定義**
2. **失敗するテストを作成** (コードがまだ存在しないため)
3. **テストを実行** し、正しい理由で失敗することを確認
4. **テストをパスさせるための最小限の実装を作成**
5. **テストを実行** し、パスすることを確認
6. **テストをグリーンのままコードをリファクタリング**
7. **カバレッジをチェック** し、80%未満の場合はテストを追加

## TDDサイクル

```
レッド → グリーン → リファクター → 繰り返し

レッド:      失敗するテストを書く
グリーン:    パスするための最小限のコードを書く
リファクター: コードを改善し、テストをパスさせ続ける
繰り返し:   次の機能/シナリオへ
```

## 使用例

```
ユーザー: /tdd 市場の流動性スコアを計算する関数が必要です

エージェント (tdd-guide):
# TDDセッション: 市場流動性スコア計算機

## ステップ1: インターフェースの定義 (スキャフォールド)

```typescript
// lib/liquidity.ts
export interface MarketData {
  totalVolume: number
  bidAskSpread: number
  activeTraders: number
  lastTradeTime: Date
}

export function calculateLiquidityScore(market: MarketData): number {
  // TODO: 実装
  throw new Error('Not implemented')
}
```

## ステップ2: 失敗するテストの作成 (レッド)

```typescript
// lib/liquidity.test.ts
import { calculateLiquidityScore } from './liquidity'

describe('calculateLiquidityScore', () => {
  it('should return high score for liquid market', () => {
    const market = {
      totalVolume: 100000,
      bidAskSpread: 0.01,
      activeTraders: 500,
      lastTradeTime: new Date()
    }

    const score = calculateLiquidityScore(market)

    expect(score).toBeGreaterThan(80)
    expect(score).toBeLessThanOrEqual(100)
  })

  it('should return low score for illiquid market', () => {
    const market = {
      totalVolume: 100,
      bidAskSpread: 0.5,
      activeTraders: 2,
      lastTradeTime: new Date(Date.now() - 86400000) // 1日前
    }

    const score = calculateLiquidityScore(market)

    expect(score).toBeLessThan(30)
    expect(score).toBeGreaterThanOrEqual(0)
  })

  it('should handle edge case: zero volume', () => {
    const market = {
      totalVolume: 0,
      bidAskSpread: 0,
      activeTraders: 0,
      lastTradeTime: new Date()
    }

    const score = calculateLiquidityScore(market)

    expect(score).toBe(0)
  })
})
```

## ステップ3: テスト実行 - 失敗の確認

```bash
npm test lib/liquidity.test.ts

FAIL lib/liquidity.test.ts
  ✕ should return high score for liquid market (2 ms)
    Error: Not implemented

1 test failed, 0 passed
```

✅ 期待通りにテストが失敗しました。実装の準備ができました。

## ステップ4: 最小限のコードを実装 (グリーン)

```typescript
// lib/liquidity.ts
export function calculateLiquidityScore(market: MarketData): number {
  // 出来高ゼロのエッジケースを処理
  if (market.totalVolume === 0) {
    return 0
  }

  // コンポーネントスコアを計算 (0-100スケール)
  const volumeScore = Math.min(market.totalVolume / 1000, 100)
  const spreadScore = Math.max(100 - (market.bidAskSpread * 1000), 0)
  const traderScore = Math.min(market.activeTraders / 10, 100)

  // 最近の活動ボーナス
  const hoursSinceLastTrade = (Date.now() - market.lastTradeTime.getTime()) / (1000 * 60 * 60)
  const recencyScore = Math.max(100 - (hoursSinceLastTrade * 10), 0)

  // 加重平均
  const score = (
    volumeScore * 0.4 +
    spreadScore * 0.3 +
    traderScore * 0.2 +
    recencyScore * 0.1
  )

  return Math.min(Math.max(score, 0), 100) // 0-100にクランプ
}
```

## ステップ5: テスト実行 - 成功の確認

```bash
npm test lib/liquidity.test.ts

PASS lib/liquidity.test.ts
  ✓ should return high score for liquid market (3 ms)
  ✓ should return low score for illiquid market (2 ms)
  ✓ should handle edge case: zero volume (1 ms)

3 tests passed
```

✅ 全てのテストが成功しました！

## ステップ6: リファクタリング (改善)

```typescript
// lib/liquidity.ts - 定数と可読性を向上させたリファクタリング
const WEIGHTS = {
  VOLUME: 0.4,
  SPREAD: 0.3,
  TRADERS: 0.2,
  RECENCY: 0.1,
} as const

const SCALE_FACTORS = {
  VOLUME: 1000,
  SPREAD: 1000,
  TRADERS: 10,
  RECENCY_PENALTY: 10,
} as const

function clamp(value: number, min: number, max: number): number {
  return Math.min(Math.max(value, min), max)
}

export function calculateLiquidityScore(market: MarketData): number {
  if (market.totalVolume === 0) return 0

  const volumeScore = Math.min(market.totalVolume / SCALE_FACTORS.VOLUME, 100)
  const spreadScore = clamp(100 - (market.bidAskSpread * SCALE_FACTORS.SPREAD), 0, 100)
  const traderScore = Math.min(market.activeTraders / SCALE_FACTORS.TRADERS, 100)

  const hoursSinceLastTrade = (Date.now() - market.lastTradeTime.getTime()) / (1000 * 60 * 60)
  const recencyScore = clamp(100 - (hoursSinceLastTrade * SCALE_FACTORS.RECENCY_PENALTY), 0, 100)

  const weightedScore =
    volumeScore * WEIGHTS.VOLUME +
    spreadScore * WEIGHTS.SPREAD +
    traderScore * WEIGHTS.TRADERS +
    recencyScore * WEIGHTS.RECENCY

  return clamp(weightedScore, 0, 100)
}
```

## ステップ7: テストがまだ成功することを確認

```bash
npm test lib/liquidity.test.ts

PASS lib/liquidity.test.ts
  ✓ should return high score for liquid market (3 ms)
  ✓ should return low score for illiquid market (2 ms)
  ✓ should handle edge case: zero volume (1 ms)

3 tests passed
```

✅ リファクタリング完了、テストはまだ成功しています！

## ステップ8: カバレッジのチェック

```bash
npm test -- --coverage lib/liquidity.test.ts

File           | % Stmts | % Branch | % Funcs | % Lines
---------------|---------|----------|---------|--------
liquidity.ts   |   100   |   100    |   100   |   100

Coverage: 100% ✅ (Target: 80%)
```

✅ TDDセッション完了！
```

## TDDベストプラクティス

**推奨:**
- ✅ 実装の前に、まずテストを書く
- ✅ 実装の前に、テストを実行して失敗することを確認する
- ✅ テストをパスさせるための最小限のコードを書く
- ✅ テストがグリーンの状態でのみリファクタリングする
- ✅ エッジケースやエラーシナリオを追加する
- ✅ 80%以上のカバレッジを目指す（クリティカルなコードは100%）

**非推奨:**
- ❌ テストの前に実装を書く
- ❌ 各変更後にテストを実行しない
- ❌ 一度に多くのコードを書きすぎる
- ❌ 失敗するテストを無視する
- ❌ 実装の詳細をテストする（振る舞いをテストする）
- ❌ すべてをモックする（統合テストを好む）

## 含めるべきテストの種類

**単体テスト** (関数レベル):
- ハッピーパスシナリオ
- エッジケース (空, null, 最大値)
- エラー条件
- 境界値

**統合テスト** (コンポーネントレベル):
- APIエンドポイント
- データベース操作
- 外部サービス呼び出し
- フックを持つReactコンポーネント

**E2Eテスト** (`/e2e` コマンドを使用):
- クリティカルなユーザージャーニー
- 複数ステップのプロセス
- フルスタックの統合

## カバレッジ要件

- **最低80%** (全てのコード)
- **100%必須**:
  - 金融計算
  - 認証ロジック
  - セキュリティクリティカルなコード
  - コアビジネスロジック

## 重要事項

**必須**: テストは実装の**前に**書かれなければなりません。TDDサイクルは:

1. **レッド** - 失敗するテストを書く
2. **グリーン** - パスするように実装する
3. **リファクター** - コードを改善する

レッドフェーズを決してスキップしないでください。テストの前にコードを書かないでください。

## 他のコマンドとの統合

- `/plan` を最初に使用して何をビルドするかを理解する
- `/tdd` を使用してテストと共に実装する
- `/build-and-fix` をビルドエラー発生時に使用する
- `/code-review` を使用して実装をレビューする
- `/test-coverage` を使用してカバレッジを検証する

## 関連エージェント

このコマンドは、以下の場所にある `tdd-guide` エージェントを呼び出します:
`~/.claude/agents/tdd-guide.md`

また、以下の場所にある `tdd-workflow` スキルを参照できます:
`~/.claude/skills/tdd-workflow/`
