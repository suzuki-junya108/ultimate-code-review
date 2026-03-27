# 自動修正プレイブック

CRITICAL/HIGH問題を自動修正するためのルールブック。
どのプロジェクトでも適用可能なユニバーサルパターンを収録。

---

## 修正の優先順位

```
CRITICAL → 即座に修正（セキュリティ, データ整合性）
HIGH     → 修正（パフォーマンス重大問題, 大きなバグ）
MEDIUM   → 警告として報告（推奨修正）
LOW      → 提案として報告（任意対応）
```

---

## セキュリティ自動修正

### 1. console.log/debugger の除去

```typescript
// 検出パターン（正規表現）
console\.(log|warn|error|debug|trace|table|info)\s*\(
debugger;

// 自動修正: 削除または構造化ロガーへ置換
// Before
console.log("debug:", data);
// After（プロジェクトにloggerがある場合）
logger.debug("Processing data", { data });
// After（loggerがない場合）
// [削除]
```

### 2. ハードコードされた機密情報

```typescript
// 検出パターン
// (?:api[_-]?key|secret|password|token|credential)\s*[:=]\s*["'][^"']{8,}["']

// Before
const API_KEY = "sk-proj-xxxxx";
const DB_PASSWORD = "admin123";

// After
const API_KEY = process.env.API_KEY;
if (!API_KEY) throw new Error("API_KEY environment variable is required");
const DB_PASSWORD = process.env.DB_PASSWORD;
if (!DB_PASSWORD) throw new Error("DB_PASSWORD environment variable is required");
```

### 3. SQLインジェクション対策

```typescript
// 検出: テンプレートリテラルや文字列連結でのSQLクエリ
// `SELECT.*\$\{` または "SELECT.*" + variable

// Before
const query = `SELECT * FROM users WHERE id = ${userId}`;
db.query("SELECT * FROM orders WHERE status = " + status);

// After
const query = "SELECT * FROM users WHERE id = ?";
db.query(query, [userId]);
// または
db.prepare("SELECT * FROM orders WHERE status = ?").bind(status);
```

### 4. 空のcatchブロック

```typescript
// 検出: } catch \(.*\) \{\s*\}  または  } catch \{.*\}

// Before
try {
  riskyOperation();
} catch (e) {}

// After
try {
  riskyOperation();
} catch (error) {
  // Intentionally suppressed: [理由を記載]
  // 理由がない場合は logger で記録
  console.error("Unexpected error in riskyOperation", error);
  throw error; // 上位に伝播
}
```

### 5. any型の使用

```typescript
// 検出: : any  または  as any  または  <any>

// Before
function process(data: any): any {
  return data;
}

// After（型が不明な場合）
function process(data: unknown): unknown {
  return data;
}

// After（Zodで検証する場合）
const DataSchema = z.object({ /* スキーマ定義 */ });
type Data = z.infer<typeof DataSchema>;
function process(rawData: unknown): Data {
  return DataSchema.parse(rawData);
}
```

---

## エラーハンドリング自動修正

### 6. 未処理のPromise

```typescript
// 検出: await なしで非同期関数を呼び出し + .catch() なし
// パターン: 行末が ; で終わる Promise 返却関数の呼び出し

// Before
someAsyncOperation();
fetchData().then(process);

// After
await someAsyncOperation();
await fetchData().then(process).catch(handleError);
```

### 7. エラーログの強化

```typescript
// 検出: catch ブロック内で error オブジェクトを使用していない

// Before
catch (error) {
  logger.error("Something went wrong");
}

// After
catch (error) {
  logger.error("Operation failed", {
    error: error instanceof Error ? {
      name: error.name,
      message: error.message,
      stack: error.stack,
    } : String(error),
    context: { /* 関連するコンテキスト */ },
  });
  throw error;
}
```

---

## 型安全自動修正

### 8. 非nullアサーション（!）の除去

```typescript
// 検出: \w+!\.  または  \w+![\[\(]

// Before
const element = document.getElementById("app")!;
const name = user!.profile!.name;

// After
const element = document.getElementById("app");
if (!element) throw new Error("Element #app not found");
const name = user?.profile?.name ?? "Unknown";
```

### 9. 型アサーション（as）の安全化

```typescript
// 検出: as \w+  （Zodなどの検証なしで使用）

// Before
const user = response.data as User;

// After（Zodがある場合）
const user = UserSchema.parse(response.data);

// After（簡易な場合）
function isUser(value: unknown): value is User {
  return typeof value === "object" && value !== null && "id" in value;
}
if (!isUser(response.data)) throw new TypeError("Invalid user data");
const user = response.data;
```

---

## パフォーマンス自動修正

### 10. マジックナンバーの定数化

```typescript
// 検出: 関数引数や条件式内の数値リテラル（0, 1以外）

// Before
if (retries > 3) { ... }
await delay(5000);
const items = data.slice(0, 20);

// After
const MAX_RETRY_COUNT = 3;
const RETRY_DELAY_MS = 5000;
const DEFAULT_PAGE_SIZE = 20;

if (retries > MAX_RETRY_COUNT) { ... }
await delay(RETRY_DELAY_MS);
const items = data.slice(0, DEFAULT_PAGE_SIZE);
```

---

## セキュリティ拡張自動修正

### 11. SSRF防止（URLバリデーション）

```typescript
// 検出: fetch/axios等のURL引数にユーザー入力が流れる
// fetch(req.body.url)  axios.get(req.query.callback)

// Before
const data = await fetch(userProvidedUrl);

// After
const ALLOWED_HOSTS = (process.env.ALLOWED_WEBHOOK_HOSTS ?? '').split(',').filter(Boolean);
const PRIVATE_IP = /^(localhost|127\.|10\.|172\.(1[6-9]|2\d|3[01])\.|192\.168\.|169\.254\.)/;
try {
  const url = new URL(userProvidedUrl);
  if (!ALLOWED_HOSTS.includes(url.hostname) || PRIVATE_IP.test(url.hostname)) {
    throw new Error(`SSRF: Forbidden host: ${url.hostname}`);
  }
  const data = await fetch(url.toString());
} catch (err) {
  if (err instanceof TypeError) throw new Error('SSRF: Invalid URL');
  throw err;
}
```

### 12. CORS wildcard → 環境変数で管理

```typescript
// 検出: cors({ origin: "*" })

// Before
app.use(cors({ origin: '*' }));

// After
const allowedOrigins = (process.env.ALLOWED_ORIGINS ?? '').split(',').filter(Boolean);
app.use(cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error(`CORS: Origin ${origin} not allowed`));
    }
  },
  credentials: true,
}));
```

### 13. セキュリティヘッダー一括設定（helmet）

```typescript
// 検出: セキュリティヘッダー設定なし
// (CSP/X-Frame-Options/HSTS等が未設定)

// Before
app.use(express.json());

// After
import helmet from 'helmet';
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"], // CSSのみ暫定的に許可
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'"],
      fontSrc: ["'self'"],
      objectSrc: ["'none'"],
      frameAncestors: ["'none'"],
      upgradeInsecureRequests: [],
    },
  },
  frameguard: { action: 'deny' },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true },
  noSniff: true,
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  permittedCrossDomainPolicies: { permittedPolicies: 'none' },
}));
```

### 14. 定時間比較（Timing Attack防止）

```typescript
// 検出: token === storedToken, apiKey === process.env.API_KEY 等

// Before
if (token === storedToken) { }

// After
import { timingSafeEqual, createHash } from 'crypto';
function safeCompare(a: string, b: string): boolean {
  try {
    // 異なる長さでもタイミングを一定にする
    const ha = createHash('sha256').update(a).digest();
    const hb = createHash('sha256').update(b).digest();
    return timingSafeEqual(ha, hb);
  } catch {
    return false;
  }
}
if (safeCompare(token, storedToken)) { }
```

### 15. Mass Assignment防止（ホワイトリスト）

```typescript
// 検出: const data = { ...req.body } または Object.assign({}, req.body)

// Before
const userData = { ...req.body };
await db.user.update({ where: { id } }, userData);

// After
// 更新可能フィールドのホワイトリスト
const UPDATABLE_USER_FIELDS = ['email', 'name', 'bio', 'avatarUrl'] as const;
type UpdatableUserField = typeof UPDATABLE_USER_FIELDS[number];
const userData = Object.fromEntries(
  UPDATABLE_USER_FIELDS
    .filter(field => field in req.body)
    .map(field => [field, req.body[field as UpdatableUserField]])
);
await db.user.update({ where: { id } }, userData);
```

### 16. Open Redirect防止

```typescript
// 検出: res.redirect(req.query.next) / res.redirect(req.body.returnUrl)

// Before
res.redirect(req.query.next as string);

// After
const ALLOWED_REDIRECT_ORIGINS = (process.env.ALLOWED_REDIRECT_ORIGINS ?? '').split(',').filter(Boolean);
function safeRedirect(res: Response, next: string | undefined, fallback = '/') {
  if (!next) return res.redirect(fallback);
  try {
    const url = new URL(next, `https://${process.env.HOST ?? 'localhost'}`);
    const isRelative = !next.startsWith('http');
    const isAllowed = ALLOWED_REDIRECT_ORIGINS.includes(url.origin);
    return res.redirect(isRelative || isAllowed ? url.pathname + url.search : fallback);
  } catch {
    return res.redirect(fallback);
  }
}
safeRedirect(res, req.query.next as string);
```

### 17. Path Traversal防止

```typescript
// 検出: fs.readFile(path.join(baseDir, userInput))

// Before
const content = await fs.readFile(path.join(uploadDir, req.params.filename), 'utf-8');

// After
import path from 'path';
const UPLOAD_DIR = path.resolve(process.env.UPLOAD_DIR ?? '/app/uploads');
const requestedPath = path.resolve(UPLOAD_DIR, req.params.filename);
// baseDir の外に出ていないか確認
if (!requestedPath.startsWith(UPLOAD_DIR + path.sep) && requestedPath !== UPLOAD_DIR) {
  throw new Error(`Path traversal detected: ${req.params.filename}`);
}
const content = await fs.readFile(requestedPath, 'utf-8');
```

### 18. Prototype Pollution防止

```typescript
// 検出: Object.assign(target, userInput) / _.merge(target, userInput)

// Before
Object.assign(config, req.body);

// After
// JSON.parse(JSON.stringify()) でプロトタイプチェーンを切断
function sanitizeInput<T>(input: unknown): T {
  if (typeof input !== 'object' || input === null) return input as T;
  return JSON.parse(JSON.stringify(input)) as T;
}
const safeBody = sanitizeInput<Record<string, unknown>>(req.body);
Object.assign(config, safeBody);
```

### 19. Input Length Validation追加

```typescript
// 検出: z.string() / z.string().optional() に最大長がない

// Before
const schema = z.object({
  username: z.string(),
  email: z.string().email(),
  comment: z.string().optional(),
});

// After
const schema = z.object({
  username: z.string().min(1).max(50),            // ユーザー名: 50文字
  email: z.string().email().max(254),              // RFC 5321 最大長
  comment: z.string().max(2000).optional(),        // コメント: 2000文字
  url: z.string().url().max(2048).optional(),      // URL: RFC 2616推奨最大長
  bio: z.string().max(500).optional(),
});
```

### 20. JWT alg:none攻撃防止

```typescript
// 検出: jwt.verify(token, secret) でalgorithmsオプションなし

// Before
const decoded = jwt.verify(token, process.env.JWT_SECRET!);

// After
const decoded = jwt.verify(token, process.env.JWT_SECRET!, {
  algorithms: ['HS256'],  // 許可アルゴリズムを明示 (none を含めない)
  issuer: process.env.JWT_ISSUER,      // 発行者の検証
  audience: process.env.JWT_AUDIENCE, // 受信者の検証
});
```

---

## インフラ自動修正

### 21. Dockerfileのrootユーザー実行修正

```dockerfile
# 検出: USER命令がない（または USER root）

# Before
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci --only=production
CMD ["node", "src/index.js"]

# After
FROM node:20-alpine
WORKDIR /app
COPY --chown=node:node . .
RUN npm ci --only=production
USER node  # node:alpineには node ユーザーが組み込みで存在
CMD ["node", "src/index.js"]
```

### 22. GitHub Actions SHA pinning

```yaml
# 検出: uses: actions/xxx@v? （タグでピン留め）

# Before
- uses: actions/checkout@v4
- uses: actions/setup-node@v4

# After（SHA でピン留め - 変更不可）
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
- uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af  # v4.1.0
# 注: pinact や Dependabot で自動更新可能
```

### 23. .env gitトラッキング警告

```bash
# 検出: .envファイルがgitで追跡されている
# git ls-files .env .env.local .env.production .env.staging

# 修正手順（[MANUAL REQUIRED]として報告）:
# 1. git rm --cached .env
# 2. echo ".env*" >> .gitignore
# 3. git commit -m "Remove .env from git tracking"
# 4. 既にプッシュされている場合: git-filter-repo でgit履歴から削除
# 5. 関連するシークレットを全て無効化・再発行
```

---

## 信頼性自動修正

### 24. グレースフルシャットダウン追加

```typescript
// 検出: process.on('SIGTERM') が未実装
// または process.on('SIGTERM', () => process.exit())

// Before
const server = app.listen(PORT, () => console.log(`Server running on ${PORT}`));

// After
const server = app.listen(PORT, () => console.log(`Server running on ${PORT}`));

let isShuttingDown = false;

const shutdown = async (signal: string) => {
  if (isShuttingDown) return;
  isShuttingDown = true;
  console.log(`${signal} received. Starting graceful shutdown...`);

  server.close(async () => {
    try {
      // 依存サービスのクローズ（プロジェクトに応じて調整）
      // await db.disconnect();
      // await redis.quit();
      console.log('Graceful shutdown complete');
      process.exit(0);
    } catch (err) {
      console.error('Error during shutdown:', err);
      process.exit(1);
    }
  });

  // K8s terminationGracePeriodSeconds (default: 30s) を超えないよう強制終了
  setTimeout(() => {
    console.error('Shutdown timeout exceeded, forcing exit');
    process.exit(1);
  }, 25000).unref();
};

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

### 25. 冪等性キーの実装

```typescript
// 検出: 支払い・メール送信・ステータス変更エンドポイントにidempotency-keyなし

// Before
app.post('/api/payments', async (req, res) => {
  const result = await processPayment(req.body);
  return res.json(result);
});

// After
app.post('/api/payments', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'] as string;
  if (!idempotencyKey) {
    return res.status(400).json({ error: 'Idempotency-Key header is required' });
  }

  // 既存の結果を確認
  const cached = await redis.get(`idempotency:payment:${idempotencyKey}`);
  if (cached) {
    return res.status(200).json(JSON.parse(cached));
  }

  const result = await processPayment(req.body);

  // 24時間キャッシュ（リトライ期間内）
  await redis.setex(`idempotency:payment:${idempotencyKey}`, 86400, JSON.stringify(result));
  return res.status(201).json(result);
});
```

---

## プライバシー自動修正

### 26. PII ログ出力サニタイズ

```typescript
// 検出: logger.xxx({ user }) や logger.xxx({ email, password })

// Before
logger.info("User action", { user, action });
logger.error("Auth failed", { email, password, token });

// After
const PII_FIELDS = new Set(['email', 'password', 'token', 'apiKey', 'secret',
  'name', 'phone', 'address', 'creditCard', 'ssn', 'dob'] as const);

function sanitizeForLog(obj: Record<string, unknown>): Record<string, unknown> {
  return Object.fromEntries(
    Object.entries(obj).map(([key, value]) => {
      const lowerKey = key.toLowerCase();
      const isPii = [...PII_FIELDS].some(f => lowerKey.includes(f));
      return [key, isPii ? '***' : value];
    })
  );
}

logger.info("User action", sanitizeForLog({ userId: user.id, action }));
logger.error("Auth failed", sanitizeForLog({ userId: user?.id }));
```

### 27. デシリアライズ後のランタイム型検証追加

```typescript
// 検出: JSON.parse(rawInput) の後にzod/validation がない

// Before
const data = JSON.parse(rawInput); // 型がunknown/anyのまま使用
await processData(data);

// After
import { z } from 'zod';

const DataSchema = z.object({
  // 受け入れるフィールドを明示的に定義
  id: z.string().uuid(),
  name: z.string().max(255),
  // ... 他フィールド
});

try {
  const parsed = JSON.parse(rawInput);
  const data = DataSchema.parse(parsed); // 型安全なランタイム検証
  await processData(data);
} catch (err) {
  if (err instanceof z.ZodError) {
    throw new ValidationError('Invalid data format', err.errors);
  }
  throw err;
}
```

### 28. SQLのDynamic ORDER BY安全化

```typescript
// 検出: `ORDER BY ${userInput}` / `ORDER BY ${req.query.sort}`

// Before
const data = await db.query(`SELECT * FROM orders ORDER BY ${req.query.sort}`);

// After
const SORTABLE_FIELDS = ['created_at', 'updated_at', 'total', 'status'] as const;
const SORT_DIRECTIONS = ['ASC', 'DESC'] as const;
type SortField = typeof SORTABLE_FIELDS[number];
type SortDirection = typeof SORT_DIRECTIONS[number];

const sortField = (SORTABLE_FIELDS as readonly string[]).includes(req.query.sort as string)
  ? req.query.sort as SortField : 'created_at';
const sortDir = (SORT_DIRECTIONS as readonly string[]).includes(req.query.dir?.toUpperCase() as string)
  ? req.query.dir?.toUpperCase() as SortDirection : 'DESC';

const data = await db.query(`SELECT * FROM orders ORDER BY ${sortField} ${sortDir}`);
```

### 29. CSP unsafe-inline除去（nonce方式）

```typescript
// 検出: script-src 'unsafe-inline' または style-src 'unsafe-inline'

// Before（Express）
res.setHeader('Content-Security-Policy', "script-src 'self' 'unsafe-inline'");

// After（nonce方式）
import { randomBytes } from 'crypto';

app.use((req, res, next) => {
  const nonce = randomBytes(16).toString('base64');
  res.locals.cspNonce = nonce;
  res.setHeader(
    'Content-Security-Policy',
    [
      `script-src 'self' 'nonce-${nonce}'`,
      "style-src 'self' 'unsafe-inline'", // CSSは暫定
      "default-src 'self'",
      "frame-ancestors 'none'",
    ].join('; ')
  );
  next();
});

// テンプレート側: <script nonce="<%= locals.cspNonce %>">...</script>
```

### 30. レート制限の追加

```typescript
// 検出: /api/login, /api/register, /api/forgot-password 等にレート制限なし

// Before
app.post('/api/login', loginHandler);

// After
import rateLimit from 'express-rate-limit';

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 10,                    // 10回まで
  standardHeaders: true,
  legacyHeaders: false,
  message: { error: 'Too many login attempts. Please try again in 15 minutes.' },
  skipSuccessfulRequests: true, // 成功したリクエストはカウントしない
});

const apiLimiter = rateLimit({
  windowMs: 60 * 1000, // 1分
  max: 100,
  standardHeaders: true,
  legacyHeaders: false,
});

app.post('/api/login', loginLimiter, loginHandler);
app.use('/api/', apiLimiter);
```

---

## 修正後の検証

自動修正を適用した後、必ず以下を実行する:

```bash
# 検証コマンド（プロジェクトに応じて適用可能なものを実行）
pnpm lint 2>/dev/null || npm run lint 2>/dev/null || eslint . 2>/dev/null || biome check . 2>/dev/null || true
pnpm typecheck 2>/dev/null || npm run typecheck 2>/dev/null || tsc --noEmit 2>/dev/null || true
pnpm test 2>/dev/null || npm test 2>/dev/null || vitest run 2>/dev/null || true
pnpm build 2>/dev/null || npm run build 2>/dev/null || true
```

検証に失敗した場合:
1. エラー内容を確認
2. 修正の意図と合致しているか判断
3. 修正を調整して再検証
4. 3回以上失敗した場合は `[MANUAL REQUIRED]` タグを付けてレポートに記載

---

## 自動修正レポート形式

```markdown
## 自動修正レポート

### 修正済み: 3件
1. [CRITICAL→FIXED] console.log除去 (src/api/handler.ts:42)
   変更: `console.log("debug:", data)` → 削除

2. [HIGH→FIXED] any型をunknown+型ガードに変換 (src/utils/parser.ts:15)
   変更: `function parse(data: any)` → `function parse(data: unknown)`

3. [HIGH→FIXED] マジックナンバーを定数化 (src/services/order.ts:88)
   変更: `if (retries > 3)` → `if (retries > MAX_RETRY_COUNT)`

### 修正不可（手動対応が必要）: 1件
1. [CRITICAL][MANUAL REQUIRED] SQL文のパラメータ化 (src/db/queries.ts:156)
   理由: クエリの構造が動的に変わるため、修正方法の判断が必要
   推奨: Drizzle/Prismaのクエリビルダーへの移行を検討
```
