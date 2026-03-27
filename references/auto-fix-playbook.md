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
