---
name: security-review
description: 認証の追加、ユーザー入力の処理、シークレットの扱い、APIエンドポイントの作成、または支払い/機密機能の実装時にこのスキルを使用します。包括的なセキュリティチェックリストとパターンを提供します。
---

# セキュリティレビュースキル

このスキルは、すべてのコードがセキュリティのベストプラクティスに従い、潜在的な脆弱性を特定することを保証します。

## アクティベートするタイミング

- 認証または認可の実装
- ユーザー入力またはファイルアップロードの処理
- 新しいAPIエンドポイントの作成
- シークレットまたは認証情報の扱い
- 支払い機能の実装
- 機密データの保存または送信
- サードパーティAPIの統合

## セキュリティチェックリスト

### 1. シークレット管理

#### ❌ 絶対にやってはいけないこと
```typescript
const apiKey = "sk-proj-xxxxx"  // ハードコードされたシークレット
const dbPassword = "password123" // ソースコード内
```

#### ✅ 常にこうすること
```typescript
const apiKey = process.env.OPENAI_API_KEY
const dbUrl = process.env.DATABASE_URL

// シークレットが存在することを確認
if (!apiKey) {
  throw new Error('OPENAI_API_KEYが設定されていません')
}
```

#### 検証ステップ
- [ ] ハードコードされたAPIキー、トークン、パスワードがない
- [ ] すべてのシークレットが環境変数にある
- [ ] `.env.local`が.gitignoreにある
- [ ] git履歴にシークレットがない
- [ ] 本番環境のシークレットがホスティングプラットフォーム（Vercel, Railway）にある

### 2. 入力検証

#### 常にユーザー入力を検証する
```typescript
import { z } from 'zod'

// 検証スキーマを定義
const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150)
})

// 処理前に検証
export async function createUser(input: unknown) {
  try {
    const validated = CreateUserSchema.parse(input)
    return await db.users.create(validated)
  } catch (error) {
    if (error instanceof z.ZodError) {
      return { success: false, errors: error.errors }
    }
    throw error
  }
}
```

#### ファイルアップロード検証
```typescript
function validateFileUpload(file: File) {
  // サイズチェック（最大5MB）
  const maxSize = 5 * 1024 * 1024
  if (file.size > maxSize) {
    throw new Error('ファイルが大きすぎます（最大5MB）')
  }

  // 型チェック
  const allowedTypes = ['image/jpeg', 'image/png', 'image/gif']
  if (!allowedTypes.includes(file.type)) {
    throw new Error('無効なファイルタイプです')
  }

  // 拡張子チェック
  const allowedExtensions = ['.jpg', '.jpeg', '.png', '.gif']
  const extension = file.name.toLowerCase().match(/\.[^.]+$/)?.[0]
  if (!extension || !allowedExtensions.includes(extension)) {
    throw new Error('無効なファイル拡張子です')
  }

  return true
}
```

#### 検証ステップ
- [ ] すべてのユーザー入力がスキーマで検証されている
- [ ] ファイルアップロードが制限されている（サイズ、型、拡張子）
- [ ] クエリでユーザー入力を直接使用していない
- [ ] ホワイトリスト検証（ブラックリストではない）
- [ ] エラーメッセージが機密情報を漏洩しない

### 3. SQLインジェクション防止

#### ❌ SQLを絶対に連結しない
```typescript
// 危険 - SQLインジェクション脆弱性
const query = `SELECT * FROM users WHERE email = '${userEmail}'`
await db.query(query)
```

#### ✅ 常にパラメータ化クエリを使用する
```typescript
// 安全 - パラメータ化クエリ
const { data } = await supabase
  .from('users')
  .select('*')
  .eq('email', userEmail)

// または生のSQLで
await db.query(
  'SELECT * FROM users WHERE email = $1',
  [userEmail]
)
```

#### 検証ステップ
- [ ] すべてのデータベースクエリがパラメータ化クエリを使用している
- [ ] SQLで文字列連結がない
- [ ] ORM/クエリビルダが正しく使用されている
- [ ] Supabaseクエリが適切にサニタイズされている

### 4. 認証と認可

#### JWTトークンの取り扱い
```typescript
// ❌ 誤り: localStorage（XSSに対して脆弱）
localStorage.setItem('token', token)

// ✅ 正しい: httpOnlyクッキー
res.setHeader('Set-Cookie',
  `token=${token}; HttpOnly; Secure; SameSite=Strict; Max-Age=3600`)
```

#### 認可チェック
```typescript
export async function deleteUser(userId: string, requesterId: string) {
  // 常に最初に認可を検証する
  const requester = await db.users.findUnique({
    where: { id: requesterId }
  })

  if (requester.role !== 'admin') {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 403 }
    )
  }

  // 削除を続行
  await db.users.delete({ where: { id: userId } })
}
```

#### 行レベルセキュリティ (Supabase)
```sql
-- すべてのテーブルでRLSを有効化
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- ユーザーは自分のデータのみ閲覧可能
CREATE POLICY "Users view own data"
  ON users FOR SELECT
  USING (auth.uid() = id);

-- ユーザーは自分のデータのみ更新可能
CREATE POLICY "Users update own data"
  ON users FOR UPDATE
  USING (auth.uid() = id);
```

#### 検証ステップ
- [ ] トークンがhttpOnlyクッキーに保存されている（localStorageではない）
- [ ] 機密操作の前に認可チェックが行われる
- [ ] SupabaseでRLSが有効になっている
- [ ] ロールベースのアクセス制御が実装されている
- [ ] セッション管理が安全である

### 5. XSS防止

#### HTMLのサニタイズ
```typescript
import DOMPurify from 'isomorphic-dompurify'

// 常にユーザー提供のHTMLをサニタイズする
function renderUserContent(html: string) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p'],
    ALLOWED_ATTR: []
  })
  return <div dangerouslySetInnerHTML={{ __html: clean }} />
}
```

#### コンテンツセキュリティポリシー
```typescript
// next.config.js
const securityHeaders = [
  {
    key: 'Content-Security-Policy',
    value: `
      default-src 'self';
      script-src 'self' 'unsafe-eval' 'unsafe-inline';
      style-src 'self' 'unsafe-inline';
      img-src 'self' data: https:;
      font-src 'self';
      connect-src 'self' https://api.example.com;
    `.replace(/\s{2,}/g, ' ').trim()
  }
]
```

#### 検証ステップ
- [ ] ユーザー提供のHTMLがサニタイズされている
- [ ] CSPヘッダーが設定されている
- [ ] 未検証の動的コンテンツレンダリングがない
- [ ] Reactの組み込みXSS保護が使用されている

### 6. CSRF保護

#### CSRFトークン
```typescript
import { csrf } from '@/lib/csrf'

export async function POST(request: Request) {
  const token = request.headers.get('X-CSRF-Token')

  if (!csrf.verify(token)) {
    return NextResponse.json(
      { error: 'Invalid CSRF token' },
      { status: 403 }
    )
  }

  // リクエストを処理
}
```

#### SameSiteクッキー
```typescript
res.setHeader('Set-Cookie',
  `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`)
```

#### 検証ステップ
- [ ] 状態を変更する操作にCSRFトークンがある
- [ ] すべてのクッキーにSameSite=Strictがある
- [ ] ダブルサブミットクッキーパターンが実装されている

### 7. レート制限

#### APIレート制限
```typescript
import rateLimit from 'express-rate-limit'

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // ウィンドウごとに100リクエスト
  message: 'Too many requests'
})

// ルートに適用
app.use('/api/', limiter)
```

#### 高負荷な操作
```typescript
// 検索に対する積極的なレート制限
const searchLimiter = rateLimit({
  windowMs: 60 * 1000, // 1分
  max: 10, // 1分あたり10リクエスト
  message: 'Too many search requests'
})

app.use('/api/search', searchLimiter)
```

#### 検証ステップ
- [ ] すべてのAPIエンドポイントにレート制限がある
- [ ] 高負荷な操作にはより厳しい制限がある
- [ ] IPベースのレート制限
- [ ] ユーザーベースのレート制限（認証済み）

### 8. 機密データの公開

#### ロギング
```typescript
// ❌ 誤り: 機密データのロギング
console.log('User login:', { email, password })
console.log('Payment:', { cardNumber, cvv })

// ✅ 正しい: 機密データを編集
console.log('User login:', { email, userId })
console.log('Payment:', { last4: card.last4, userId })
```

#### エラーメッセージ
```typescript
// ❌ 誤り: 内部詳細の公開
catch (error) {
  return NextResponse.json(
    { error: error.message, stack: error.stack },
    { status: 500 }
  )
}

// ✅ 正しい: 一般的なエラーメッセージ
catch (error) {
  console.error('Internal error:', error)
  return NextResponse.json(
    { error: 'An error occurred. Please try again.' },
    { status: 500 }
  )
}
```

#### 検証ステップ
- [ ] ログにパスワード、トークン、シークレットがない
- [ ] ユーザー向けのエラーメッセージが一般的である
- [ ] 詳細なエラーはサーバーログにのみ記録される
- [ ] ユーザにスタックトレースが公開されない

### 9. ブロックチェーンセキュリティ (Solana)

#### ウォレット検証
```typescript
import { verify } from '@solana/web3.js'

async function verifyWalletOwnership(
  publicKey: string,
  signature: string,
  message: string
) {
  try {
    const isValid = verify(
      Buffer.from(message),
      Buffer.from(signature, 'base64'),
      Buffer.from(publicKey, 'base64')
    )
    return isValid
  } catch (error) {
    return false
  }
}
```

#### トランザクション検証
```typescript
async function verifyTransaction(transaction: Transaction) {
  // 受取人を検証
  if (transaction.to !== expectedRecipient) {
    throw new Error('Invalid recipient')
  }

  // 金額を検証
  if (transaction.amount > maxAmount) {
    throw new Error('Amount exceeds limit')
  }

  // ユーザーが十分な残高を持っているか検証
  const balance = await getBalance(transaction.from)
  if (balance < transaction.amount) {
    throw new Error('Insufficient balance')
  }

  return true
}
```

#### 検証ステップ
- [ ] ウォレット署名が検証されている
- [ ] トランザクション詳細が検証されている
- [ ] トランザクション前に残高チェックが行われる
- [ ] ブラインド署名がない

### 10. 依存関係のセキュリティ

#### 定期的な更新
```bash
# 脆弱性をチェック
npm audit

# 自動修正可能な問題を修正
npm audit fix

# 依存関係を更新
npm update

# 古いパッケージをチェック
npm outdated
```

#### ロックファイル
```bash
# 常にロックファイルをコミット
git add package-lock.json

# CI/CDで再現可能なビルドに使用
npm ci  # npm install の代わりに
```

#### 検証ステップ
- [ ] 依存関係が最新である
- [ ] 既知の脆弱性がない（npm auditがクリーン）
- [ ] ロックファイルがコミットされている
- [ ] GitHubでDependabotが有効になっている
- [ ] 定期的なセキュリティアップデート

## セキュリティテスト

### 自動セキュリティテスト
```typescript
// 認証をテスト
test('requires authentication', async () => {
  const response = await fetch('/api/protected')
  expect(response.status).toBe(401)
})

// 認可をテスト
test('requires admin role', async () => {
  const response = await fetch('/api/admin', {
    headers: { Authorization: `Bearer ${userToken}` }
  })
  expect(response.status).toBe(403)
})

// 入力検証をテスト
test('rejects invalid input', async () => {
  const response = await fetch('/api/users', {
    method: 'POST',
    body: JSON.stringify({ email: 'not-an-email' })
  })
  expect(response.status).toBe(400)
})

// レート制限をテスト
test('enforces rate limits', async () => {
  const requests = Array(101).fill(null).map(() =>
    fetch('/api/endpoint')
  )

  const responses = await Promise.all(requests)
  const tooManyRequests = responses.filter(r => r.status === 429)

  expect(tooManyRequests.length).toBeGreaterThan(0)
})
```

## デプロイ前セキュリティチェックリスト

本番デプロイの前には必ず:

- [ ] **シークレット**: ハードコードされたシークレットがなく、すべて環境変数にある
- [ ] **入力検証**: すべてのユーザー入力が検証されている
- [ ] **SQLインジェクション**: すべてのクエリがパラメータ化されている
- [ ] **XSS**: ユーザーコンテンツがサニタイズされている
- [ ] **CSRF**: 保護が有効になっている
- [ ] **認証**: 適切なトークン処理
- [ ] **認可**: ロールチェックが実施されている
- [ ] **レート制限**: すべてのエンドポイントで有効
- [ ] **HTTPS**: 本番環境で強制
- [ ] **セキュリティヘッダー**: CSP, X-Frame-Optionsが設定されている
- [ ] **エラー処理**: エラーに機密データが含まれていない
- [ ] **ロギング**: 機密データがログに記録されていない
- [ ] **依存関係**: 最新で、脆弱性がない
- [ ] **行レベルセキュリティ**: Supabaseで有効
- [ ] **CORS**: 適切に設定されている
- [ ] **ファイルアップロード**: 検証済み（サイズ、型）
- [ ] **ウォレット署名**: 検証済み（ブロックチェーンの場合）

## リソース

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Next.js Security](https://nextjs.org/docs/security)
- [Supabase Security](https://supabase.com/docs/guides/auth)
- [Web Security Academy](https://portswigger.net/web-security)

---

**忘れないで**: セキュリティは任意ではありません。1つの脆弱性がプラットフォーム全体を危険にさらす可能性があります。疑問がある場合は、慎重を期してください。
