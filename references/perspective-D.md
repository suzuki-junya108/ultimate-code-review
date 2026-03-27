# カテゴリD: 深層技術品質（視点20-25）

循環的複雑度・データモデル・設定管理・並行処理・API設計・オンボーディングに踏み込んだ深層分析視点。

---

## 視点20: 複雑度分析

### 循環的複雑度（Cyclomatic Complexity）

```
測定方法: 分岐点（if/else/switch/while/for/&&/||/??/?.）の数 + 1

目安:
  1-10: 低複雑度 ✅ 理解・テストが容易
  11-20: 中複雑度 🟡 テストケースが多い
  21-50: 高複雑度 🔴 リファクタリング推奨
  50+:   超高複雑度 ⛔ 即座にリファクタリング必要
```

### 複雑度の高い関数を検出する指標

```
1. 長さ: 50行を超える関数
2. ネスト深度: 4レベルを超えるif/try/forの入れ子
3. 条件分岐数: 1関数内にif/switchが10以上
4. 引数数: 引数が5以上の関数（オブジェクト化を検討）
5. 返り値の多様性: 関数が3種類以上の異なる型/形式を返す
6. ロジックの混在: DB操作 + ビジネスロジック + HTTP処理が同一関数
```

- [ ] 循環的複雑度が20を超える関数がないか
- [ ] ネストが4レベルを超えていないか（早期リターンで削減可能か）
- [ ] 1つの関数が複数の責務を持っていないか
- [ ] 長いswitch/if-elseチェーンをポリモーフィズムやマップに置き換えられないか

### 認知的複雑度（Cognitive Complexity）

循環的複雑度より「人間が理解しにくさ」に特化した指標:
- ネストが深いほどコストが高い（線形ではなく指数的）
- 入れ子の制御フローは追加ペナルティ
- 連続した同様の操作（Array.map().filter()）はコスト低

```typescript
// 認知的複雑度が高い例
function process(data) {
  if (data) {              // +1
    for (const item of data) {  // +2（ネスト内）
      if (item.active) {   // +3（ネスト内）
        if (!item.deleted) { // +4（ネスト内）
          try {
            // ...
          } catch (e) {    // +5（ネスト内）
            if (e.type === "A") { // +6（ネスト内）
              // ...
            }
          }
        }
      }
    }
  }
} // 認知的複雑度: 21

// 改善例: 早期リターン + 関数抽出
function process(data) {
  if (!data) return;
  data.filter(isActiveItem).forEach(processItem);
}
function isActiveItem(item) { return item.active && !item.deleted; }
function processItem(item) { /* 個別処理 */ }
```

---

## 視点21: データモデル設計

### スキーマ設計の評価

```sql
-- 検出: データ型の不適切な選択
CREATE TABLE orders (
  total VARCHAR(20), -- 金額をVARCHARで保存 → 計算が困難
  created_date VARCHAR(20), -- 日時をVARCHARで保存 → ソート・比較が困難
  status INT, -- ステータスをINTで保存 → 意味が不明
);

-- 推奨
CREATE TABLE orders (
  total DECIMAL(10, 2), -- 金額はDECIMAL
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP, -- 日時はDATETIME
  status TEXT CHECK(status IN ('pending', 'confirmed', 'cancelled')), -- ENUMまたはCHECK制約
);
```

- [ ] データ型が適切か（金額はDECIMAL, 日時はDATETIME/TIMESTAMP）
- [ ] NULLを許容するフィールドが適切か（不必要なNULLは複雑さを増す）
- [ ] インデックスが適切に設定されているか

### 正規化の評価

```
第一正規形（1NF）違反の検出:
  - カンマ区切りの値を1フィールドに格納
  - 配列をVARCHARに格納 (例: "tag1,tag2,tag3")

第三正規形（3NF）違反の検出:
  - 関数従属: user.company_nameがuser.company_idに依存するのにordersテーブルに含まれる
  - 更新異常の可能性

非正規化が許容されるケース（明示的な設計判断が必要）:
  - パフォーマンス最適化（読み取り重視）
  - イベントソーシング（不変ログ）
  - レポートテーブル
```

- [ ] 繰り返しグループがないか（1NF）
- [ ] 不必要なデータ重複がないか（3NF）
- [ ] 非正規化がある場合、それが意図的かつドキュメント化されているか

### マイグレーションの安全性

```sql
-- 危険: データ消失の可能性
ALTER TABLE users DROP COLUMN email; -- 本番データを削除
ALTER TABLE orders MODIFY amount INT NOT NULL; -- NULLデータが存在する場合失敗

-- 危険: ロングロック（大テーブルで)
ALTER TABLE orders ADD COLUMN notes TEXT; -- 全テーブルロック（MySQLの一部バージョン）

-- 推奨: 後方互換なマイグレーション（Blue-Greenデプロイ対応）
-- Step 1: 新カラムを追加（NULLable）
ALTER TABLE users ADD COLUMN username VARCHAR(255);
-- Step 2: データを埋める
UPDATE users SET username = email WHERE username IS NULL;
-- Step 3: NOT NULL制約を追加（データ確認後）
ALTER TABLE users MODIFY username VARCHAR(255) NOT NULL;
```

- [ ] マイグレーションがロールバック可能か
- [ ] 大テーブルへのDDLが長時間ロックを引き起こさないか
- [ ] 既存データが壊れないか（NULLableにしてからデフォルト値を設定等）
- [ ] マイグレーションとアプリコードが同時デプロイでも安全か

---

## 視点22: 設定管理

### 環境ごとの設定分離

```
検出すべき問題:
1. 環境変数が検証されていない
   const API_URL = process.env.API_URL; // undefinedの可能性

2. 機密情報がコードにハードコード
   const SECRET_KEY = "my-secret"; // 危険

3. 開発用の設定が本番に流入
   const DB_URL = "localhost:5432"; // 本番でも動く場合がある

4. 設定の型安全がない
   const timeout = parseInt(process.env.TIMEOUT); // NaNの可能性
```

```typescript
// 推奨: 起動時に設定を検証
import { z } from "zod";

const EnvSchema = z.object({
  API_KEY: z.string().min(1, "API_KEY is required"),
  DATABASE_URL: z.string().url("DATABASE_URL must be a valid URL"),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  NODE_ENV: z.enum(["development", "staging", "production"]).default("development"),
});

export const config = EnvSchema.parse(process.env);
```

- [ ] すべての必須環境変数が起動時に検証されているか
- [ ] 環境変数に型安全な変換が行われているか
- [ ] `.env.example` が最新の状態か（実際の `.env` との同期）
- [ ] 機密情報がコードリポジトリに含まれていないか

### 設定の階層管理

- [ ] デフォルト値が合理的か（本番で安全なデフォルト）
- [ ] 環境別の設定が明確に分離されているか（dev/staging/production）
- [ ] 設定変更がコード変更なしにできるか

---

## 視点23: 並行処理・非同期パターン（詳細）

視点19（運用）の補足として、コードレベルの詳細分析を行う。

### 非同期パターンの評価

```typescript
// アンチパターン1: Promise Hell（コールバック地獄の非同期版）
fetchUser()
  .then(user => fetchOrders(user.id)
    .then(orders => fetchItems(orders[0].id)
      .then(items => { /* 深いネスト */ })));

// 推奨: async/await でフラット化
const user = await fetchUser();
const orders = await fetchOrders(user.id);
const items = await fetchItems(orders[0].id);

// アンチパターン2: 直列実行できる非依存処理
const user = await fetchUser(id); // 待つ必要ない
const config = await fetchConfig(); // 待つ必要ない
// → Promise.all([fetchUser(id), fetchConfig()]) で並列化

// アンチパターン3: 適切でない並列化
await Promise.all(
  thousands.map(item => db.insert(item)) // DBコネクションが枯渇
);
// → バッチサイズを制限（pLimit等のライブラリ使用）
```

### トランザクション管理

```typescript
// 危険: トランザクションなしの複数更新
await db.updateOrder(orderId, { status: "confirmed" });
await db.createShipment(orderId); // ここで失敗するとorderとshipmentが不整合

// 推奨: トランザクションで原子性を保証
await db.transaction(async (tx) => {
  await tx.updateOrder(orderId, { status: "confirmed" });
  await tx.createShipment(orderId);
}); // 失敗時は自動ロールバック
```

- [ ] 複数のDB更新操作がトランザクションで囲まれているか
- [ ] トランザクションのネストが適切に管理されているか
- [ ] デッドロック回避のためにリソース取得順序が一貫しているか

---

## 視点24: API設計原則

### RESTful API評価

```
HTTPメソッドの正しい使用:
  GET: 読み取り専用（副作用なし）
  POST: 新規作成
  PUT: 完全更新（全フィールド）
  PATCH: 部分更新（一部フィールド）
  DELETE: 削除

検出: HTTPメソッドの誤用
  POST /api/getUser → GET /api/users/:id に変更
  GET /api/deleteUser?id=1 → DELETE /api/users/:id に変更
  POST /api/updateUser → PUT/PATCH /api/users/:id に変更
```

```
URLの設計原則:
  良い例:
    GET /api/v1/orders → 注文一覧
    GET /api/v1/orders/:id → 個別注文
    POST /api/v1/orders → 注文作成
    PATCH /api/v1/orders/:id → 注文更新
    DELETE /api/v1/orders/:id → 注文削除

  悪い例:
    GET /api/getOrders → 動詞を使っている
    POST /api/order/delete → GETに削除操作
    GET /api/user_orders_list → スネークケース混在
```

- [ ] HTTPメソッドが適切に使用されているか
- [ ] URLが名詞（リソース）ベースになっているか
- [ ] APIがバージョン管理されているか（/v1/, /v2/）
- [ ] クエリパラメータが適切に使用されているか（フィルタ, ソート, ページネーション）

### エラーレスポンスの標準化

```typescript
// 推奨: 一貫したエラーレスポンス形式
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "The requested user does not exist",
    "details": { "userId": "123" }
  }
}

// 避けるべき: HTTPステータスと一致しないレスポンス
// HTTP 200 OK + { "success": false, "error": "not found" }
```

- [ ] エラーレスポンスの形式が一貫しているか
- [ ] HTTPステータスコードが適切か（200, 201, 400, 401, 403, 404, 422, 500）
- [ ] クライアントが必要な情報を含むエラーメッセージか

---

## 視点25: オンボーディング品質

### 新規開発者が理解できるか

```
新しいチームメンバーが以下を30分以内にできるか：
1. ローカル環境のセットアップ
2. テストの実行
3. 最初のコード変更
4. 変更の確認方法の把握
```

- [ ] READMEにセットアップ手順が明確に書かれているか
- [ ] 必要な環境変数が `.env.example` に記載されているか
- [ ] 開発サーバーの起動方法が明確か（`pnpm dev` 等）
- [ ] テストの実行方法が明確か（`pnpm test` 等）

### コードの自己説明性

```typescript
// 悪い: 理解に外部知識が必要
const NEApiBase = "https://api.next-engine.org/api_v1/";
await nec.r_login(uid, state, code); // 引数の意味が不明

// 良い: コードから意図が読める
const NEXT_ENGINE_API_BASE = "https://api.next-engine.org/api_v1/";
await nextEngineClient.refreshLoginToken({
  uid: userId,
  state: oauthState,
  accessCode: code,
});
```

- [ ] 関数/変数名から意図が明確に読み取れるか
- [ ] 外部システム固有の仕様（特殊なAPI仕様等）がコメントで説明されているか
- [ ] 複雑なビジネスロジックに「なぜ」のコメントがあるか
- [ ] プロジェクト固有の略語・用語が定義されているか（ドキュメントまたはコードコメント）

### ローカル開発環境

- [ ] 外部サービスなしでローカルで実行できるか（Docker/モック等）
- [ ] 開発用のシードデータが整備されているか
- [ ] デバッグのやりやすさが考慮されているか（ログが読みやすい等）
- [ ] よくある問題とその解決方法がドキュメント化されているか（TROUBLESHOOTING）
