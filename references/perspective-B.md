# カテゴリB: 開発者体験（視点7-13）

開発者が安心して変更・拡張できるかを評価する視点。
アーキテクチャ・コード品質・テスタビリティ・依存関係・ドキュメント・保守性・型設計の各視点を網羅した詳細チェックリスト。

---

## 視点7: アーキテクチャ適合性

### レイヤーアーキテクチャの検証

```
典型的なレイヤー構成:
  Presentation (API/UI)
      ↓
  Service (ビジネスロジック)
      ↓
  Repository (データアクセス)
      ↓
  Infrastructure (外部依存)

検出すべき違反:
- Controller/Route から Repository を直接呼ぶ（Service をバイパス）
- Service から HTTP/リクエスト情報に依存
- Repository にビジネスロジックが混入
- 循環依存（A→B→C→A）
```

- [ ] 責務が適切なレイヤーに配置されているか
- [ ] 依存関係の方向が一貫しているか（上位から下位のみ）
- [ ] 循環依存がないか
- [ ] DIP（依存性逆転原則）が守られているか（上位が下位の具体実装に依存していない）

### SOLID原則の完全チェック

#### 単一責任原則（SRP: Single Responsibility Principle）

- [ ] クラス/モジュールは1つの責任のみを持つか
- [ ] 関数は1つのことだけを行うか
- [ ] ファイルサイズが300行を超えていないか（超える場合は分割を検討）

#### 開放/閉鎖原則（OCP: Open/Closed Principle）

```typescript
// 違反: 新しい型を追加するたびにif/elseが増える
function calculateDiscount(type: string, price: number): number {
  if (type === "student") return price * 0.8;
  if (type === "senior") return price * 0.7;
  if (type === "employee") return price * 0.5;
  // 新しい割引タイプ追加のたびにこの関数を変更する必要がある
  return price;
}

// 推奨: 拡張に対して開いている（既存コードを変更せずに追加可能）
interface DiscountStrategy { apply(price: number): number; }
class StudentDiscount implements DiscountStrategy { apply = (p) => p * 0.8; }
class SeniorDiscount implements DiscountStrategy { apply = (p) => p * 0.7; }
// 新割引 → 新クラスを追加するだけ。既存コード変更不要
```

- [ ] 新しい種別・タイプの追加が既存コードの変更なしにできるか（Strategy/Polymorphism）
- [ ] 長いif-else/switchチェーンが「新機能追加で必ず変更が必要」になっていないか

#### リスコフ置換原則（LSP: Liskov Substitution Principle）

```typescript
// 違反: サブクラスが親クラスの契約を破る
class Rectangle { setWidth(w: number) {} setHeight(h: number) {} }
class Square extends Rectangle {
  setWidth(w: number) { this.width = this.height = w; } // 高さも変わる！
  // Square はRectangleを置き換えられない → LSP違反
}

// チェック: サブクラスが親クラスのテストケースを全て通過するか
// → 「IS-A」より「CAN-DO-WHAT」で継承の妥当性を判断
```

- [ ] サブクラスが親クラスの事前条件・事後条件・不変条件を守っているか
- [ ] 継承よりコンポジションを優先しているか（過度な継承階層に注意）
- [ ] `instanceof` チェックが多用されていないか（LSP違反のサイン）

#### インターフェース分離原則（ISP: Interface Segregation Principle）

```typescript
// 違反: 巨大インターフェース（使わないメソッドへの依存を強制）
interface Worker {
  work(): void;
  eat(): void;  // ロボットには不要
  sleep(): void; // ロボットには不要
}

// 推奨: 最小限のインターフェース分割
interface Workable { work(): void; }
interface Eatable { eat(): void; }
interface Sleepable { sleep(): void; }
class HumanWorker implements Workable, Eatable, Sleepable { ... }
class Robot implements Workable { ... } // 不要なメソッドを実装しなくてよい
```

- [ ] インターフェースが過度に大きくないか（未使用メソッドを実装させていないか）
- [ ] クライアントが使わないメソッドに依存させられていないか

#### 依存性逆転原則（DIP: Dependency Inversion Principle）

```typescript
// 違反: 上位モジュールが下位実装に依存
class OrderService {
  private db = new MySQLDatabase(); // 具体実装に直接依存 → テスト不可
}

// 推奨: 上位モジュールは抽象に依存
interface IDatabase { query(sql: string): Promise<unknown>; }
class OrderService {
  constructor(private db: IDatabase) {} // 抽象に依存 → テスト可能
}
```

- [ ] Service層がRepositoryの具体実装ではなくインターフェースに依存しているか
- [ ] DIコンテナや依存注入パターンが適切に使われているか

### 既存パターンとの一貫性

- [ ] プロジェクトの既存コードと同じパターンを使用しているか
- [ ] 新しいパターンを導入する場合、既存コードを置き換える計画があるか
- [ ] `.claude/patterns.md` や `.claude/decisions.md` のルールに準拠しているか（あれば）

---

## 視点8: コード品質

### 命名の明確さ

```
検出: 意味の不明な名前
- data, tmp, item, obj → 具体的な名前に変更
- d, r, x （1文字変数） → ループ変数(i,j,k)以外は禁止
- doStuff(), processData() → 何をするかが不明

検出: 誤解を招く名前
- isActive が状態変更を行う関数名（Principle of Least Surprise違反）
- getUserData が保存も行う関数名
```

- [ ] 変数・関数・クラス名から意図が読み取れるか
- [ ] 略語・頭字語が適切に使われているか（プロジェクト全体で一貫）
- [ ] ネガティブ条件名がないか（isNotLoggedIn より isLoggedIn の否定が推奨）

### コードの重複（DRY）

```typescript
// 検出: 3箇所以上の同一ロジック
const userA = { ...raw, createdAt: new Date(raw.created_at) };
const userB = { ...raw, createdAt: new Date(raw.created_at) };
const userC = { ...raw, createdAt: new Date(raw.created_at) };
// → mapRawToUser() 関数に抽出

// 検出: コピー&ペーストされた似たような関数
function validateEmail(email) { /* 20行 */ }
function validatePhone(phone) { /* ほぼ同じ20行 */ }
// → validate(value, type) の汎用関数に統合
```

- [ ] 同じロジックが3箇所以上に存在しないか
- [ ] マジックナンバーが定数化されているか
- [ ] 似た処理が統合できる可能性がないか

### 複雑度の管理

```
早期リターンパターンの適用:
// 悪い例
function process(user) {
  if (user) {
    if (user.isActive) {
      if (user.hasPermission) {
        // ... 本体
      }
    }
  }
}

// 良い例
function process(user) {
  if (!user) return;
  if (!user.isActive) return;
  if (!user.hasPermission) return;
  // ... 本体（ネスト削減）
}
```

- [ ] ネストが4レベルを超えていないか
- [ ] 早期リターンで複雑さを削減できる箇所がないか
- [ ] switch/if-elseチェーンが長すぎないか（戦略パターン・ポリモーフィズムを検討）

---

## 視点9: テスタビリティ

### テストカバレッジの評価

```bash
# テストファイルの存在確認
find . -name "*.test.*" -o -name "*.spec.*" | grep -v node_modules

# カバレッジレポートの確認（あれば）
cat coverage/summary.txt 2>/dev/null || cat coverage/lcov-report/index.html 2>/dev/null | head -50
```

- [ ] 新規追加された主要なビジネスロジックにテストがあるか
- [ ] テストがないコードが変更された場合、テストが追加されたか
- [ ] バグ修正に再現テストが含まれているか

### テストの質

```typescript
// 悪いテスト: 実装の詳細をテスト
it("should call repository.save once", () => {
  expect(repo.save).toHaveBeenCalledTimes(1); // 脆い
});

// 良いテスト: 振る舞いをテスト
it("should persist the user when valid data is provided", async () => {
  const user = await userService.create({ name: "Alice", email: "alice@example.com" });
  expect(user.id).toBeDefined();
  expect(user.name).toBe("Alice");
});
```

- [ ] テストが「実装の詳細」ではなく「振る舞い」をテストしているか
- [ ] テスト名から何をテストしているかが明確か
- [ ] テストが独立して実行できるか（テスト間に依存がない）
- [ ] フレイキーテスト（不安定なテスト）がないか
- [ ] エッジケース・境界値のテストがあるか

### テストピラミッド比率の評価

```
理想的なテストピラミッド:
  E2E/UI テスト:        10% — 遅い・壊れやすい、クリティカルフローのみ
  統合テスト:           20% — DBや外部APIとの結合
  ユニットテスト:       70% — 速い・安定・豊富

アンチパターン: テストアイスクリームコーン（逆ピラミッド）
  E2Eテストが多い → CI が遅い、フレイキーになりやすい
  ユニットテストが少ない → 変更の影響範囲が分からない

```bash
# テスト種別の比率確認
find . -name "*.e2e.*" -o -name "*.e2e-spec.*" | grep -v node_modules | wc -l
find . -name "*.spec.*" -o -name "*.test.*" | grep -v node_modules | grep -v ".e2e." | wc -l
```

- [ ] テストピラミッドの比率が適切か（ユニット > 統合 > E2E）
- [ ] E2Eテストがクリティカルフロー（ログイン, 購入, 重要ユーザージャーニー）に集中しているか

### Contract Testing（マイクロサービス/API利用時）

```typescript
// Pact.js を使用したConsumer-Driven Contract Test
// Consumer（呼び出し側）がProvider（提供側）への期待を定義
describe("User Service Contract", () => {
  it("should return user object", () => {
    provider.addInteraction({
      state: "user 1 exists",
      uponReceiving: "a GET request for user 1",
      withRequest: { method: "GET", path: "/users/1" },
      willRespondWith: {
        status: 200,
        body: { id: like(1), name: like("Alice") },
      },
    });
    // → Provider側はこの契約を満たすことをCIで検証する必要がある
  });
});
```

- [ ] マイクロサービス間/外部APIとの契約テスト（Pact等）が実装されているか（マイクロサービス構成の場合）
- [ ] APIスキーマの変更（OpenAPI/gRPC proto）が下位互換性を維持しているか検証されているか

### Mutation Testing（テストの有効性確認）

```
Mutation Testing（突然変異テスト）の概念:
  コードに意図的なバグ（突然変異）を挿入し、テストがそれを検出できるか検証する
  - a > b → a >= b に変更
  - return true → return false に変更
  - db.save() を削除

  突然変異を検出できないテスト → テストがコードの振る舞いを保証していない

Stryker（JavaScript向け）の活用:
  npx stryker run 2>/dev/null
  目標: Mutation Score > 70%（それ以下はテストが薄い可能性）
```

- [ ] テストカバレッジが高くても内容が薄くないか（`if(true)` をテストするだけ等）
- [ ] 重要なビジネスロジックにMutation Testingを適用する価値があるか（Stryker推奨）

### モックの適切な使用

- [ ] 外部依存（DB, API, ファイルシステム）が適切にモック化されているか
- [ ] 過度なモックで実際の挙動が隠蔽されていないか
- [ ] テスト用のフィクスチャ/ファクトリが整備されているか

---

## 視点10: 依存関係管理

### 新規依存の評価

```
新規パッケージ追加時のチェック:
1. 週間ダウンロード数 > 10,000（npm registry確認）
2. 最終更新が1年以内
3. TypeScript型定義あり（@types/xxx または組み込み）
4. セキュリティ脆弱性なし（pnpm audit / npm audit）
5. ライセンスがプロジェクトと互換（MIT, Apache-2.0等）
6. バンドルサイズが許容範囲（bundlephobia等で確認）
```

- [ ] 追加された依存関係は必要最小限か
- [ ] 20行以下で実装できる機能をライブラリで代替していないか
- [ ] 同様の機能をもつライブラリがすでに依存関係に存在しないか

### 依存関係のセキュリティ

```bash
# 脆弱性チェック
pnpm audit --audit-level=high 2>/dev/null || npm audit --audit-level=high 2>/dev/null

# 古いパッケージの確認
pnpm outdated 2>/dev/null || npm outdated 2>/dev/null
```

- [ ] 高・重大な脆弱性を持つ依存関係がないか
- [ ] ピン留め（exact version）されているか（セキュリティが重要な依存）
- [ ] lock ファイルが最新か

### 内部依存関係

- [ ] パッケージ間の依存方向が設計に従っているか（apps → packages のみ）
- [ ] 循環依存がないか
- [ ] 共有パッケージが過度に肥大化していないか

---

## 視点11: ドキュメンテーション

### コードとドキュメントの同期

- [ ] READMEと実装が一致しているか（古い手順が残っていないか）
- [ ] API仕様書（あれば）と実際のAPIが一致しているか
- [ ] 複雑なビジネスロジックにコメント（「なぜ」の説明）があるか
- [ ] 非自明な副作用や前提条件が文書化されているか

### コメントの質

```typescript
// 悪い: コードの翻訳
// i を 1 増やす
i++;

// 悪い: 古い情報
// TODO: これはバグ（2022年のコメント、バグはとっくに修正済み）

// 良い: 意図と理由
// レート制限を超えないよう、Shopify APIは1秒あたり2リクエストまで
await delay(500);

// 良い: 非自明な仕様
// NextEngineは在庫数が0の場合、nullを返すことがある（0ではなくnull）
const stock = item.stock ?? 0;
```

- [ ] コメントが「何をするか」ではなく「なぜするか」を説明しているか
- [ ] コメントが最新の実装と一致しているか
- [ ] 公開APIに適切なJSDocがあるか（必要な場合）

---

## 視点12: 保守性

### 変更の局所化

- [ ] ビジネスルールの変更が1つの場所で完結するか
- [ ] 設定値が1箇所に集約されているか（マジックナンバーが散在していないか）
- [ ] 機能フラグが整理されているか（不要になったフラグが残っていないか）

### デッドコードの検出

```typescript
// 検出パターン
// 1. 呼び出されていない関数/クラス
export function unusedUtil() { ... } // exportされているが未使用

// 2. 到達不可能なコード
if (condition) {
  return result;
  doSomething(); // この行は実行されない
}

// 3. 常にtrueまたはfalseの条件
if (process.env.FEATURE_FLAG === "false") { // "false"文字列は常にtruthy
  enableFeature();
}
```

- [ ] 呼び出されていない関数・変数・コードパスがないか
- [ ] コメントアウトされたコードが残っていないか
- [ ] import されているが使用されていないモジュールがないか

### 後方互換性

- [ ] 公開インターフェース（API, エクスポートされた型）が変更されていないか
- [ ] 変更がある場合、移行パスが提供されているか
- [ ] 破壊的変更がある場合、バージョンが上げられているか（semver）

---

## 視点13: 型設計

### TypeScript固有の品質チェック

```typescript
// 危険: any型の使用
const data: any = fetchData(); // 型安全を破壊
function process(input: any): any { ... }

// 危険: 過度な型アサーション
const user = data as User; // 検証なし → ランタイムエラーの可能性
const el = document.querySelector("#app") as HTMLElement; // null可能性を無視

// 危険: 非nullアサーション
const name = user!.name; // ランタイムエラーの可能性

// 推奨: Zodでの型検証
const UserSchema = z.object({ id: z.string(), name: z.string() });
type User = z.infer<typeof UserSchema>;
const user = UserSchema.parse(rawData); // 型安全かつランタイム安全
```

- [ ] `any` 型の使用がないか（`unknown` + 型ガードを使用）
- [ ] 型アサーション（`as`）が検証なしで使用されていないか
- [ ] 非nullアサーション（`!`）が安全に使用されているか
- [ ] ジェネリクスが適切に使用され、型安全性を高めているか
- [ ] 型定義がドメインモデルを正確に表現しているか
- [ ] 境界値（外部データ）でZodなどのランタイム検証があるか

### 型の再利用性

- [ ] 重複した型定義がないか
- [ ] 型パラメータが過度に複雑になっていないか
- [ ] ユーティリティ型（Partial, Required, Pick, Omit）が効果的に使われているか
