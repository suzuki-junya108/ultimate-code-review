# カテゴリA: ユーザー価値（視点1-6）

エンドユーザーとビジネスに直接影響を与える視点。
セキュリティ・パフォーマンス・アクセシビリティ・国際化・エラーハンドリングの各視点を網羅した詳細チェックリスト。

---

## 視点1: 機能的正確性

### チェックリスト

**ビジネスロジックの正確性**
- [ ] 要件・仕様書と実装が一致しているか（READMEやdocs/design/を確認）
- [ ] 境界値（0, -1, null, 空文字, 最大値）が正しく処理されるか
- [ ] 状態遷移が正確か（注文ステータス、ユーザー状態など）
- [ ] 計算ロジックに誤りがないか（四捨五入、切り捨て、通貨計算の精度）
- [ ] 複数の条件が組み合わさった場合に正しく動作するか

**エッジケース**
- [ ] リスト操作: 空配列, 要素1つ, 重複要素, null要素
- [ ] 文字列処理: 空文字, 改行含む, Unicode特殊文字, 超長文字列
- [ ] 数値処理: ゼロ除算, オーバーフロー, 浮動小数点誤差
- [ ] 日時処理: タイムゾーン, 夏時間, うるう年, エポック
- [ ] 並行処理: 同時実行, レース条件, デッドロック

**データ整合性**
- [ ] トランザクション境界が適切か（部分更新が起きないか）
- [ ] 冪等性が必要な操作（支払い, メール送信）で保証されているか
- [ ] 外部キー制約・ユニーク制約との整合性

---

## 視点2: セキュリティ（OWASP Top 10）

### A01:2021 アクセス制御の不備

```typescript
// 検出: 認可チェックなし
app.get("/admin/users", handler); // 権限チェックなし
app.get("/users/:id", (req) => getUser(req.params.id)); // 所有者チェックなし

// 検出: IDOR（Insecure Direct Object Reference）
// URLパラメータのIDを直接使用し、アクセス権限チェックがない
```

- [ ] すべての保護されたルートに認証・認可ミドルウェアがあるか
- [ ] ユーザーは自分のリソースのみアクセスできるか（IDOR防止）
- [ ] 管理者機能は管理者ロールのみアクセスできるか
- [ ] 水平権限昇格・垂直権限昇格の可能性がないか

### A02:2021 暗号化の失敗

- [ ] 機密データ（パスワード, APIキー, PII）が平文で保存・転送されていないか
- [ ] 弱い暗号アルゴリズム（MD5, SHA1）が使われていないか
- [ ] TLS/HTTPSが必要な箇所で使用されているか
- [ ] ハードコードされた秘密情報がないか

```typescript
// 危険
const API_KEY = "sk-proj-xxxxx"; // ハードコード
const hash = md5(password); // 弱いハッシュ

// 安全
const API_KEY = process.env.API_KEY;
const hash = await bcrypt.hash(password, 12);
```

### A03:2021 インジェクション

```typescript
// SQL インジェクション検出
`SELECT * FROM users WHERE id = ${userId}` // 危険
db.query("SELECT * FROM users WHERE id = " + id) // 危険

// コマンドインジェクション検出
exec(`ls ${userInput}`) // 危険
child_process.exec(command) // 危険（execFile を使うこと）

// NoSQL インジェクション
db.find({ $where: userInput }) // 危険
```

- [ ] SQLクエリはすべてパラメータ化されているか
- [ ] コマンド実行にユーザー入力が含まれていないか
- [ ] テンプレートエンジンでのエスケープは適切か

### A07:2021 認証の不備

```typescript
// 危険
if (password === storedPassword) // 平文比較
const token = Math.random().toString() // 弱いトークン
```

- [ ] パスワードはbcrypt等の安全なハッシュで保存されているか
- [ ] セッショントークンは暗号学的に安全なランダム値か
- [ ] JWTの署名検証が正しく行われているか
- [ ] パスワードリセットトークンの有効期限は適切か

### A05:2021 セキュリティ設定ミス

- [ ] CORS設定が `*` になっていないか
- [ ] セキュリティヘッダー（CSP, X-Frame-Options, HSTS）が設定されているか
- [ ] デバッグ情報が本番で露出していないか
- [ ] エラーメッセージに内部情報（スタックトレース, DB情報）が含まれていないか

### 機密情報の管理

```typescript
// 危険: ログへの機密情報出力
console.log("User:", { email, password }) // 危険
logger.info("Auth", { token, apiKey }) // 危険

// 危険: レスポンスへの内部情報
res.json({ error: error.stack }) // 危険
```

- [ ] ログに機密情報（パスワード, トークン, APIキー, PII）が出力されていないか
- [ ] エラーレスポンスに内部実装の詳細が含まれていないか

### CSRF（Cross-Site Request Forgery）

```typescript
// 検出: 状態変更エンドポイントにCSRF保護がない
app.post('/api/transfer', handler); // CSRFトークン検証なし
app.put('/api/users/:id', handler); // SameSiteチェックなし
```

- [ ] POST/PUT/PATCH/DELETE エンドポイントにCSRFトークン検証があるか（Cookie認証使用時）
- [ ] Cookieに `SameSite=Strict` または `SameSite=Lax` が設定されているか
- [ ] `X-CSRF-Token` ヘッダーの検証があるか
- [ ] Authorizationヘッダーのみ使用（Cookieless）の場合はCSRF不要であることを確認

### Clickjacking防止

- [ ] `X-Frame-Options: DENY` ヘッダーが設定されているか
- [ ] CSP の `frame-ancestors 'none'` ディレクティブがあるか
- [ ] 上記いずれかで iframe埋め込みが防止されているか

### セキュリティヘッダー完全チェック

必須ヘッダーの確認:

```
Content-Security-Policy:
  - unsafe-inline / unsafe-eval が含まれていないか
  - default-src が 'self' に限定されているか
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

- [ ] 上記6ヘッダーが全て設定されているか（helmet.js等）
- [ ] CSPに `unsafe-inline` または `unsafe-eval` が使われていないか

### A08:2021 ソフトウェア・データ整合性の不具合

```typescript
// 危険: ユーザー入力を直接評価
eval(userInput);
Function(userInput)();
require(dynamicPath);  // パス操作可能

// 危険: デシリアライズ後に型検証なし
const data = JSON.parse(rawInput); // 型がunknownのまま使用

// 危険: 外部CDNからSRIハッシュなしで読み込み
<script src="https://cdn.example.com/lib.js"></script>
```

- [ ] `eval()` / `Function()` にユーザー入力が流れていないか
- [ ] `require(dynamicPath)` でパスにユーザー入力が混入していないか
- [ ] `JSON.parse(userInput)` 後にzod等のランタイム型検証があるか
- [ ] 外部CDNスクリプトにSubResource Integrity (SRI) ハッシュがあるか

### A09:2021 セキュリティログおよびモニタリングの失敗

- [ ] 認証失敗（ログイン失敗、不正アクセス試行）がログに記録されているか
- [ ] セキュリティイベント（権限エラー、レート制限超過）がモニタリングされているか
- [ ] ログが改ざん防止されているか（append-only、外部転送）
- [ ] 重大なセキュリティイベントにアラートが設定されているか

### A10:2021 SSRF（Server-Side Request Forgery）

```typescript
// 危険: ユーザー入力をURLとして使用
const data = await fetch(req.body.url);
const result = await axios.get(req.query.callback);
const webhook = await http.get(userProvidedEndpoint);

// 特に危険: クラウドメタデータエンドポイント
// http://169.254.169.254/latest/meta-data/ → AWSクレデンシャル取得可能
```

- [ ] `fetch` / `axios` / `http.get` 等のURL引数にユーザー入力が流れていないか
- [ ] Webhookエンドポイントのドメイン検証があるか（allowlist方式）
- [ ] プライベートIPアドレス（10.x, 172.16-31.x, 192.168.x, 169.254.x, localhost）への接続を防止しているか

### Mass Assignment脆弱性

```typescript
// 危険: req.body を直接DBに渡す
const user = { ...req.body };
await db.user.create(user); // isAdmin: true が混入可能

// 危険: ORMの全フィールド更新
await User.update(req.body, { where: { id } }); // role等が混入可能

// 推奨: ホワイトリスト方式
const allowedFields = ['email', 'name', 'bio'];
const safeData = pick(req.body, allowedFields);
await db.user.update(safeData, { where: { id } });
```

- [ ] `req.body` を直接DB操作に渡していないか
- [ ] 更新可能フィールドがホワイトリストで制限されているか
- [ ] TypeScriptの型キャストだけでは防げないことを認識しているか（`req.body as UpdateUserDTO`）

### Open Redirect

```typescript
// 危険
res.redirect(req.query.next as string);
res.redirect(req.body.returnUrl);
window.location.href = userProvidedUrl;
```

- [ ] リダイレクト先URLにユーザー入力が使われていないか
- [ ] 使う場合は origin が allowlist に含まれるか検証しているか
- [ ] 相対パスのみ許可する制限があるか

### ReDoS（正規表現サービス拒否攻撃）

```typescript
// 危険: 指数時間の正規表現（Catastrophic Backtracking）
/^(a+)+$/  // "aaaaaaaaab"で指数時間
/(a|aa)+$/  // 類似のパターン
/^([a-zA-Z0-9]+([._]?[a-zA-Z0-9]+)*@...)+$/  // 複雑なメール検証

// 危険: ユーザー入力を正規表現として使用
const userRegex = new RegExp(userInput); // ReDoS攻撃の入口
```

- [ ] ユーザー入力が `new RegExp()` に渡されていないか
- [ ] 量化子の入れ子（`(a+)+`, `(a|aa)+`等）がないか
- [ ] 長い入力でもタイムアウトしない正規表現か（`safe-regex` ライブラリで検証推奨）

### DOM-based XSS（フロントエンド）

```typescript
// 危険: ユーザー入力をDOMに直接挿入
element.innerHTML = userInput;
element.outerHTML = userInput;
document.write(userInput);

// フレームワーク固有の危険パターン
<div dangerouslySetInnerHTML={{ __html: content }} />  // React
<div v-html="content"></div>  // Vue
```

- [ ] `innerHTML` / `outerHTML` / `document.write` にユーザー入力が流れていないか
- [ ] `dangerouslySetInnerHTML` の使用箇所はDOMPurifyでサニタイズされているか
- [ ] `v-html` の使用箇所はサニタイズされているか

### タイミング攻撃（Timing Attack）

```typescript
// 危険: 通常の文字列比較（文字一致するほど時間が長くなる）
if (token === storedToken) { }
if (apiKey === process.env.API_KEY) { }
if (signature === expectedSignature) { }

// 安全: 定時間比較
import { timingSafeEqual } from 'crypto';
const match = timingSafeEqual(
  Buffer.from(providedToken),
  Buffer.from(storedToken)
);
```

- [ ] トークン/署名の比較に `crypto.timingSafeEqual()` を使っているか
- [ ] パスワード比較に `bcrypt.compare()` を使っているか
- [ ] ユーザー存在確認でタイミング差が生じていないか（"User not found" vs "Wrong password"）

### JWT固有攻撃

```typescript
// 危険: アルゴリズム指定なし（alg:none 攻撃可能）
jwt.verify(token, secret);

// 危険: JKU/X5U ヘッダーの検証なし
// 攻撃者が独自の公開鍵URLを指定してトークン検証を迂回可能

// 安全:
jwt.verify(token, publicKey, {
  algorithms: ['RS256'],  // none を含めない、許可アルゴリズムを明示
});
```

- [ ] `jwt.verify()` の `algorithms` オプションで許可アルゴリズムを明示しているか
- [ ] `"none"` アルゴリズムを許可していないか
- [ ] JWTの秘密鍵が十分に長く、ランダムか（`"secret"` や `"password"` でないか）
- [ ] JKU/X5U ヘッダーの検証があるか（使用する場合）

### OWASP API Security Top 10（API特有）

**API3: Broken Object Property Level Authorization（過剰なデータ公開）**
```typescript
// 危険: DBオブジェクトをそのままレスポンス
const user = await db.user.findById(id);
res.json(user); // password, internalHash, role等が漏洩

// 安全: 必要フィールドのみ返す
const { id, name, email } = user;
res.json({ id, name, email });
```

- [ ] APIレスポンスに内部フィールド（ハッシュ値、内部ID、権限情報）が含まれていないか
- [ ] GraphQLでintrospectionが本番環境で無効化されているか

**API4: Unrestricted Resource Consumption**
```typescript
// 危険: ページング上限なし
const items = await db.items.findAll({ limit: req.query.limit }); // 無制限

// 安全: 最大値を強制
const limit = Math.min(Number(req.query.limit) || 20, 100);
const items = await db.items.findAll({ limit });
```

- [ ] 一覧取得APIにページング上限があるか（最大100件等）
- [ ] ファイルアップロードのサイズ制限があるか
- [ ] リクエストボディのサイズ制限があるか

**API5: Broken Function Level Authorization**
- [ ] 全HTTPメソッド（GET/POST/PUT/PATCH/DELETE）に権限チェックがあるか
- [ ] 同一リソースで、メソッドによって権限チェックが漏れていないか

**API9: Improper Inventory Management（シャドーAPI）**
- [ ] 古いAPIバージョン（/v1/）が廃止後も残っていないか
- [ ] ドキュメントにないエンドポイント（シャドーAPI）がないか
- [ ] デバッグ用エンドポイントが本番に残っていないか

---

## 視点3: パフォーマンス

### データベース・クエリ最適化

```
N+1クエリパターンの検出:
for (const order of orders) {
  const user = await db.user.findById(order.userId); // N+1!
}
→ 修正: JOIN / include / eager loading を使用

不必要なSELECT *:
const users = await db.query("SELECT * FROM users");
→ 修正: 必要なカラムのみ選択
```

- [ ] N+1クエリが発生していないか（ループ内でのDB呼び出し）
- [ ] 不必要なフルスキャンが発生していないか
- [ ] 適切なインデックスが設定されているか（WHERE, ORDER BY, JOIN カラム）
- [ ] バルク操作が可能な箇所で個別操作していないか

### キャッシュ戦略

- [ ] 頻繁にアクセスされる不変データがキャッシュされているか
- [ ] キャッシュの無効化戦略が適切か
- [ ] 分散環境でのキャッシュ整合性は保たれるか

### メモリ・CPU

```
検出パターン:
- 巨大なデータセットをメモリに全ロード → ストリーム処理に変更
- 同期的な重い処理がメインスレッドをブロック → 非同期化
- 不要なオブジェクト生成のループ → 使い回しを検討
```

- [ ] 大量データ処理でストリーム/ページネーションを使用しているか
- [ ] メモリリークの可能性がある（クローズされないリソース）がないか
- [ ] 非同期処理で不要なブロッキングがないか

### フロントエンド固有（Webアプリのみ）

- [ ] バンドルサイズが大きくなっていないか（dynamic import / code splitting）
- [ ] 不必要な再レンダリングが発生していないか（useMemo/useCallback の適切な使用）
- [ ] 画像・メディアの最適化はされているか

---

## 視点4: アクセシビリティ（Webフロントエンドのみ）

**APIプロジェクトの場合**: このセクションはスキップ

### WCAG 2.1 AA基準

- [ ] すべての画像に適切なalt属性があるか
- [ ] フォームの入力フィールドにlabelが関連付けられているか
- [ ] キーボードのみで操作できるか（Tab/Shift+Tab/Enter/Space）
- [ ] フォーカスが視覚的に確認できるか（focus-visible スタイル）
- [ ] 色のコントラスト比が基準（AA: 4.5:1）を満たしているか
- [ ] ARIAロールが適切に使用されているか（ボタン, ダイアログ, ナビゲーション）
- [ ] スクリーンリーダーで意味が伝わるか

### エラーメッセージのアクセシビリティ

- [ ] エラーメッセージが関連するフィールドに紐付いているか
- [ ] エラーが発生したことを色以外の方法でも伝えているか
- [ ] エラーメッセージが具体的で修正方法を示しているか

---

## 視点5: 国際化（i18n）

**国際化が要件にないプロジェクト**: このセクションはスキップ

### ハードコードされたテキスト

```typescript
// 危険: ハードコードされた文字列
<button>Submit</button>
alert("Error occurred");
return "User not found";

// 推奨
<button>{t("submit")}</button>
alert(t("error.occurred"));
return t("user.notFound");
```

- [ ] ユーザーに表示されるすべてのテキストが翻訳可能か
- [ ] 日付・時刻がロケールに応じてフォーマットされるか
- [ ] 通貨がロケールに応じてフォーマットされるか（例: ¥, $, €）
- [ ] 文字列の連結ではなく、メッセージフォーマットを使用しているか
- [ ] 複数形（1件, 2件以上）が適切に処理されているか
- [ ] RTL（右から左）言語のレイアウトが考慮されているか（該当する場合）

---

## 視点6: エラーハンドリング（ユーザー体験）

### エラーの分類と伝達

```
ユーザーエラー（4xx）:
  - 明確なメッセージ + 修正方法を提示
  - 技術的な詳細を含めない
  例: "メールアドレスが無効です。xxx@example.com の形式で入力してください。"

システムエラー（5xx）:
  - 親しみやすいメッセージ + サポート連絡先
  - ログにフルスタックトレース
  - ユーザーにIDを提供（問い合わせ用）
  例: "一時的なエラーが発生しました（ID: ABC123）。しばらくお待ちください。"

ネットワークエラー:
  - オフライン時のフォールバック
  - 自動リトライ（適切な間隔）
  - ユーザーへの状態通知
```

- [ ] エラーメッセージはユーザーにとって意味のある内容か
- [ ] 修正方法が明示されているか（バリデーションエラー）
- [ ] すべてのエラーパスが処理されているか（ネットワーク, タイムアウト, 権限）
- [ ] エラーが発生した後にユーザーが回復できるか（フォームデータ保持など）
- [ ] ローディング状態・エラー状態・空状態のUIが実装されているか
- [ ] エラーログに診断に必要な情報が含まれているか（コンテキスト, リクエストID）

### 非同期エラーハンドリング

```typescript
// 危険: 未処理のPromise rejection
fetchData().then(process); // catchなし
someAsyncFunction(); // awaitなし

// 危険: 空のcatchブロック
try {
  await riskyOperation();
} catch (e) {} // エラーを握りつぶす

// 推奨
try {
  const data = await fetchData();
  await process(data);
} catch (error) {
  logger.error("Failed to process data", { error, context });
  // ユーザーへの通知またはリトライ
}
```

- [ ] すべてのPromiseにエラーハンドリングがあるか
- [ ] 空のcatchブロックがないか
- [ ] グローバルエラーハンドラーが設定されているか（unhandledRejection）
