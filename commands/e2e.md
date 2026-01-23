---
description: Playwrightでエンドツーエンドテストを生成・実行します。テストジャーニーの作成、テストの実行、スクリーンショット/ビデオ/トレースのキャプチャ、アーティファクトのアップロードを行います。
---

# E2Eコマンド

このコマンドは **e2e-runner** エージェントを呼び出し、Playwrightを使用してエンドツーエンドテストを生成、維持、実行します。

## このコマンドの機能

1. **テストジャーニーの生成** - ユーザーフローのためのPlaywrightテストを作成します
2. **E2Eテストの実行** - ブラウザ間でテストを実行します
3. **アーティファクトのキャプチャ** - 失敗時にスクリーンショット、ビデオ、トレースを取得します
4. **結果のアップロード** - HTMLレポートとJUnit XMLをアップロードします
5. **不安定なテストの特定** - 不安定なテストを隔離します

## 使用する場面

`/e2e` は以下の場合に使用します:
- 重要なユーザージャーニー（ログイン、取引、支払い）のテスト
- 複数ステップのフローがエンドツーエンドで機能することの確認
- UIインタラクションとナビゲーションのテスト
- フロントエンドとバックエンドの統合の検証
- 本番デプロイの準備

## 仕組み

e2e-runnerエージェントは以下を実行します:

1. **ユーザーフローの分析** とテストシナリオの特定
2. **Playwrightテストの生成** (Page Object Modelパターン使用)
3. **テストの実行** (Chrome, Firefox, Safariなど複数ブラウザ)
4. **失敗のキャプチャ** (スクリーンショット、ビデオ、トレース)
5. **レポートの生成** (結果とアーティファクト)
6. **不安定なテストの特定** と修正の推奨

## 使用例

```
ユーザー: /e2e 市場検索と表示フローをテストして

エージェント (e2e-runner):
# E2Eテスト生成: 市場検索と表示フロー

## 特定されたテストシナリオ

**ユーザージャーニー:** 市場検索 → 結果表示 → 市場クリック → 詳細表示

**テストカバレッジ:**
1. 市場ページに移動
2. セマンティック検索を実行
3. 検索結果を検証
4. 最初の結果をクリック
5. 市場詳細ページが読み込まれることを検証
6. チャートがレンダリングされることを検証

## 生成されたテストコード

```typescript
// tests/e2e/markets/search-and-view.spec.ts
import { test, expect } from '@playwright/test'
import { MarketsPage } from '../../pages/MarketsPage'
import { MarketDetailsPage } from '../../pages/MarketDetailsPage'

test.describe('Market Search and View Flow', () => {
  test('user can search markets and view details', async ({ page }) => {
    // 1. 市場ページに移動
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    // ページ読み込みを検証
    await expect(page).toHaveTitle(/Markets/)
    await expect(page.locator('h1')).toContainText('Markets')

    // 2. セマンティック検索を実行
    await marketsPage.searchMarkets('election')

    // APIレスポンスを待機
    await page.waitForResponse(resp =>
      resp.url().includes('/api/markets/search') && resp.status() === 200
    )

    // 3. 検索結果を検証
    const marketCards = marketsPage.marketCards
    await expect(marketCards.first()).toBeVisible()
    const resultCount = await marketCards.count()
    expect(resultCount).toBeGreaterThan(0)

    // 検索結果のスクリーンショットを撮る
    await page.screenshot({ path: 'artifacts/search-results.png' })

    // 4. 最初の結果をクリック
    const firstMarketTitle = await marketCards.first().textContent()
    await marketCards.first().click()

    // 5. 市場詳細ページが読み込まれることを検証
    await expect(page).toHaveURL(/\/markets\/[a-z0-9-]+/)

    const detailsPage = new MarketDetailsPage(page)
    await expect(detailsPage.marketName).toBeVisible()
    await expect(detailsPage.marketDescription).toBeVisible()

    // 6. チャートがレンダリングされることを検証
    await expect(detailsPage.priceChart).toBeVisible()

    // 市場名が一致することを確認
    const detailsTitle = await detailsPage.marketName.textContent()
    expect(detailsTitle?.toLowerCase()).toContain(
      firstMarketTitle?.toLowerCase().substring(0, 20) || ''
    )

    // 市場詳細のスクリーンショットを撮る
    await page.screenshot({ path: 'artifacts/market-details.png' })
  })

  test('search with no results shows empty state', async ({ page }) => {
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    // 存在しない市場を検索
    await marketsPage.searchMarkets('xyznonexistentmarket123456')

    // 空の状態を検証
    await expect(page.locator('[data-testid="no-results"]')).toBeVisible()
    await expect(page.locator('[data-testid="no-results"]')).toContainText(
      /no.*results|no.*markets/i
    )

    const marketCount = await marketsPage.marketCards.count()
    expect(marketCount).toBe(0)
  })

  test('can clear search and see all markets again', async ({ page }) => {
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    // 初期の市場数
    const initialCount = await marketsPage.marketCards.count()

    // 検索を実行
    await marketsPage.searchMarkets('trump')
    await page.waitForLoadState('networkidle')

    // フィルタリングされた結果を検証
    const filteredCount = await marketsPage.marketCards.count()
    expect(filteredCount).toBeLessThan(initialCount)

    // 検索をクリア
    await marketsPage.searchInput.clear()
    await page.waitForLoadState('networkidle')

    // すべての市場が再度表示されることを検証
    const finalCount = await marketsPage.marketCards.count()
    expect(finalCount).toBe(initialCount)
  })
})
```

## テストの実行

```bash
# 生成されたテストを実行
npx playwright test tests/e2e/markets/search-and-view.spec.ts

Running 3 tests using 3 workers

  ✓  [chromium] › search-and-view.spec.ts:5:3 › user can search markets and view details (4.2s)
  ✓  [chromium] › search-and-view.spec.ts:52:3 › search with no results shows empty state (1.8s)
  ✓  [chromium] › search-and-view.spec.ts:67:3 › can clear search and see all markets again (2.9s)

  3 passed (9.1s)

Artifacts generated:
- artifacts/search-results.png
- artifacts/market-details.png
- playwright-report/index.html
```

## テストレポート

```
╔══════════════════════════════════════════════════════════════╗
║                    E2Eテスト結果                             ║
╠══════════════════════════════════════════════════════════════╣
║ ステータス: ✅ 全テスト成功                                  ║
║ 合計:       3 テスト                                         ║
║ 成功:       3 (100%)                                         ║
║ 失敗:       0                                                ║
║ 不安定:     0                                                ║
║ 期間:       9.1s                                             ║
╚══════════════════════════════════════════════════════════════╝

アーティファクト:
📸 スクリーンショット: 2 ファイル
📹 ビデオ: 0 ファイル (失敗時のみ)
🔍 トレース: 0 ファイル (失敗時のみ)
📊 HTMLレポート: playwright-report/index.html

レポート表示: npx playwright show-report
```

✅ E2EテストスイートはCI/CD統合の準備ができました！
```

## テストアーティファクト

テスト実行時に、以下のアーティファクトがキャプチャされます:

**全テストで:**
- タイムラインと結果を含むHTMLレポート
- CI統合用のJUnit XML

**失敗時のみ:**
- 失敗状態のスクリーンショット
- テストのビデオ録画
- デバッグ用のトレースファイル（ステップバイステップ再生）
- ネットワークログ
- コンソールログ

## アーティファクトの表示

```bash
# HTMLレポートをブラウザで表示
npx playwright show-report

# 特定のトレースファイルを表示
npx playwright show-trace artifacts/trace-abc123.zip

# スクリーンショットは artifacts/ ディレクトリに保存されます
open artifacts/search-results.png
```

## 不安定なテストの検出

テストが断続的に失敗する場合:

```
⚠️  不安定なテストを検出: tests/e2e/markets/trade.spec.ts

テストは 10回中7回成功 (70% 成功率)

一般的な失敗:
"Timeout waiting for element '[data-testid="confirm-btn"]'"

推奨される修正:
1. 明示的な待機を追加: await page.waitForSelector('[data-testid="confirm-btn"]')
2. タイムアウトを増やす: { timeout: 10000 }
3. コンポーネントの競合状態を確認
4. 要素がアニメーションで隠れていないか確認

隔離の推奨: 修正されるまで test.fixme() としてマーク
```

## ブラウザ設定

テストはデフォルトで複数のブラウザで実行されます:
- ✅ Chromium (Desktop Chrome)
- ✅ Firefox (Desktop)
- ✅ WebKit (Desktop Safari)
- ✅ Mobile Chrome (任意)

ブラウザを調整するには `playwright.config.ts` で設定します。

## CI/CD統合

CIパイプラインに追加します:

```yaml
# .github/workflows/e2e.yml
- name: Install Playwright
  run: npx playwright install --with-deps

- name: Run E2E tests
  run: npx playwright test

- name: Upload artifacts
  if: always()
  uses: actions/upload-artifact@v3
  with:
    name: playwright-report
    path: playwright-report/
```

## PMX固有のクリティカルフロー

PMXでは、これらのE2Eテストを優先します:

**🔴 クリティカル (常に成功する必要がある):**
1. ユーザーはウォレットを接続できる
2. ユーザーは市場を閲覧できる
3. ユーザーは市場を検索できる（セマンティック検索）
4. ユーザーは市場の詳細を表示できる
5. ユーザーは取引を発注できる（テスト資金で）
6. 市場が正しく解決される
7. ユーザーは資金を引き出すことができる

**🟡 重要:**
1. 市場作成フロー
2. ユーザープロフィールの更新
3. リアルタイムの価格更新
4. チャートのレンダリング
5. 市場のフィルタリングとソート
6. モバイルレスポンシブレイアウト

## ベストプラクティス

**推奨:**
- ✅ 保守性のためにPage Object Modelを使用する
- ✅ セレクタにはdata-testid属性を使用する
- ✅ 任意のタイムアウトではなく、APIレスポンスを待つ
- ✅ 重要なユーザージャーニーをエンドツーエンドでテストする
- ✅ mainにマージする前にテストを実行する
- ✅ テストが失敗したときはアーティファクトを確認する

**非推奨:**
- ❌ 壊れやすいセレクタを使用する（CSSクラスは変更される可能性がある）
- ❌ 実装の詳細をテストする
- ❌ 本番環境に対してテストを実行する
- ❌ 不安定なテストを無視する
- ❌ 失敗時のアーティファクトレビューをスキップする
- ❌ すべてのエッジケースをE2Eでテストする（単体テストを使用）

## 重要事項

**PMXにとってクリティカル:**
- 実際のお金が関わるE2Eテストは、テストネット/ステージングのみで実行する必要があります
- 取引テストを本番環境に対して実行しないでください
- 金融テストには `test.skip(process.env.NODE_ENV === 'production')` を設定してください
- 少額のテスト資金を持つテストウォレットのみを使用してください

## 他のコマンドとの統合

- `/plan` を使用してテストする重要なジャーニーを特定
- `/tdd` を使用して単体テスト（より速く、より詳細）
- `/e2e` を使用して統合およびユーザージャーニーテスト
- `/code-review` を使用してテストの品質を検証

## 関連エージェント

このコマンドは、以下の場所にある `e2e-runner` エージェントを呼び出します:
`~/.claude/agents/e2e-runner.md`

## クイックコマンド

```bash
# 全てのE2Eテストを実行
npx playwright test

# 特定のテストファイルを指定して実行
npx playwright test tests/e2e/markets/search.spec.ts

# ヘッダーモードで実行（ブラウザ表示）
npx playwright test --headed

# テストをデバッグ
npx playwright test --debug

# テストコードを生成
npx playwright codegen http://localhost:3000

# レポートを表示
npx playwright show-report
```
