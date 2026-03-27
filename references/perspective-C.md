# カテゴリC: 運用・成長（視点14-19）

本番稼働とシステムの継続的な健全性に関わる視点。
技術的負債・可観測性・CI/CD・ライセンスコンプライアンス・スケーラビリティ・後方互換性・並行処理の各視点を網羅した詳細チェックリスト。

---

## 視点14: 技術的負債

### 負債の識別と定量化

```bash
# TODO/FIXME/HACK/XXX の数を計測
grep -rn "TODO\|FIXME\|HACK\|XXX" src/ --include="*.ts" --include="*.tsx" \
  | grep -v node_modules | wc -l

# Issue番号なしの負債マーカー（放置リスクが高い）
grep -rn "TODO\|FIXME" src/ --include="*.ts" \
  | grep -v "#[0-9]\+" | head -20
```

- [ ] TODO/FIXMEコメントが放置されていないか（issue番号がないものは特に注意）
- [ ] 技術的負債が意図的か偶発的かを区別できるか
- [ ] 負債が「今は許容するが後で解消する」という合意のもとで導入されているか

### 負債パターンの検出

```
コードの匂い（Code Smells）:
1. 神クラス（God Object）: 1000行超のクラス、20以上のメソッド
2. 長すぎるメソッド: 50行超の関数
3. ショットガンサージェリー: 小さな変更で多数のファイル変更が必要
4. 特性の横恋慕（Feature Envy）: クラスが他クラスのデータを頻繁に使う
5. スパゲッティコード: 深いネスト、複雑な条件分岐の絡み合い
6. コピー&ペースト: 同一またはほぼ同一のコードが複数箇所に
```

- [ ] コードの匂いがないか
- [ ] リファクタリングが完了した際に関連する負債マーカーが削除されているか
- [ ] 負債が週単位で増加しているか（進化適応分析の結果を参照）

---

## 視点15: 可観測性（Observability）

### ログの品質評価

```typescript
// 悪い: 情報不足
logger.error("Error occurred"); // 原因不明
catch (e) {} // エラーを握りつぶし

// 悪い: 機密情報の漏洩
logger.info("Login attempt", { email, password }); // パスワードが露出

// 良い: 構造化ログ + 十分なコンテキスト
logger.error("Failed to process order", {
  errorCode: "ORDER_PROCESS_001",
  error: { name: error.name, message: error.message, stack: error.stack },
  context: { orderId, userId, action: "processOrder" },
  requestId: ctx.requestId,
  timestamp: new Date().toISOString(),
});
```

- [ ] ログに十分なコンテキスト情報が含まれているか（requestId, userId, action）
- [ ] ログレベルが適切か（error: 即時対応要, warn: 注意要, info: 通常操作）
- [ ] 機密情報（パスワード, トークン, APIキー, PII）がログに出力されていないか
- [ ] エラーにエラーコードが割り当てられているか（後から検索できるか）

### メトリクスとトレーシング

- [ ] 重要な操作の所要時間が記録されているか
- [ ] エラー率が追跡可能か
- [ ] 外部API呼び出しにトレースID（相関ID）が伝播しているか
- [ ] ヘルスチェックエンドポイントが存在するか（`/health`）

### OpenTelemetry / メトリクス品質

```typescript
// 推奨: OpenTelemetry 3本柱の整合
// Metrics: 集計された計測値（Prometheus形式）
// Traces: リクエスト処理の因果関係（Jaeger/Zipkin）
// Logs: 構造化ログ（trace_idと紐付け）

// Prometheus 命名規則の確認:
// ✅ http_requests_total        （カウンター: _total サフィックス）
// ✅ request_duration_seconds   （ヒストグラム: _seconds/_bytes）
// ✅ active_connections         （ゲージ: サフィックスなし）
// ❌ httpRequests, requestDuration（camelCase は不可）
// ❌ response_time_ms（_ms より _seconds を推奨）
```

**Label Cardinality（ラベル爆発）の検出:**
```typescript
// 危険: 高カーディナリティ ラベル（時系列が爆発する）
histogram.record(duration, {
  user_id: userId,     // ユーザー数分の時系列が生成される → OOM
  request_id: reqId,   // リクエストごとに一意 → 最悪パターン
});

// 推奨: 低カーディナリティ ラベルのみ
histogram.record(duration, {
  status_code: "200",  // 有限の値
  route: "/api/orders", // 有限の値
  method: "GET",       // 有限の値
});
```

- [ ] Prometheusメトリクス名がsnake_case + 適切なサフィックス（_total/_seconds/_bytes）か
- [ ] ラベルにuser_id/request_id等の高カーディナリティ値が含まれていないか
- [ ] OpenTelemetryのTrace IDがログに伝播されているか（ログ⇔トレースの紐付け）
- [ ] 外部API呼び出しにSpanが設定されているか（分散トレーシング）

### アラートの設定

- [ ] 重大なエラー（5xx, 認証失敗の急増）にアラートが設定されているか
- [ ] アラートが「アクション可能」か（鳴ったら何をすべきか明確か）
- [ ] アラート疲れ防止: フラッピング（頻繁なON/OFF）を抑制する設定があるか
- [ ] アラートに「何をすべきか」のRunbookリンクが含まれているか
- [ ] 症状ベースアラート（SLO違反）が原因ベースアラートより優先されているか
- [ ] オンコールの担当者がアラートを受け取れるか

---

## 視点16: CI/CD統合

### パイプラインの品質

```yaml
# 最低限必要なCI/CDステップ
- lint: コードスタイルの強制
- typecheck: TypeScriptの型チェック
- test: ユニット/統合テスト
- build: ビルド成功確認
- security-scan: 依存関係の脆弱性スキャン
```

- [ ] CI/CDパイプラインがすべての変更に対して実行されるか
- [ ] lint, typecheck, test, buildが全てパスしているか
- [ ] セキュリティスキャン（依存関係の脆弱性）がCIに含まれるか
- [ ] デプロイが自動化されているか（手動操作が最小か）

### デプロイの安全性

- [ ] ロールバック手順が明確か
- [ ] マイグレーションが後方互換か（デプロイ中も旧バージョンが動作するか）
- [ ] フィーチャーフラグで段階的リリースできるか（必要な場合）
- [ ] ステージング環境でテストされているか

### ブランチ保護

- [ ] mainブランチへの直接プッシュが制限されているか
- [ ] PRにレビュアーが設定されているか
- [ ] CI/CDが失敗した場合にマージがブロックされるか

### GitHub Actions セキュリティ（.github/workflows/ 存在時）

```yaml
# 危険: タグはミュータブル（改ざんされる可能性がある）
uses: actions/checkout@v4

# 推奨: コミットSHAにピン留め（イミュータブル）
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

# 危険: 過剰な権限
permissions: write-all

# 推奨: 最小権限
permissions:
  contents: read
  pull-requests: write  # 必要なもののみ

# 危険: GITHUB_TOKENを第三者に渡す可能性
run: curl -H "Authorization: ${{ secrets.GITHUB_TOKEN }}" ${{ github.event.inputs.url }}
# → ユーザー入力URL（SSRF）+ トークン漏洩の組み合わせ
```

- [ ] サードパーティアクションがコミットSHA（フルハッシュ）にピン留めされているか
- [ ] GITHUB_TOKENのpermissionsが最小権限（各jobに明示的に設定）か
- [ ] ワークフロー内にsecretsがハードコードされていないか（`echo "$SECRET"` 等）
- [ ] pull_request_target トリガーが安全に使用されているか（フォークからの悪意あるPR）
- [ ] self-hosted runnerが使用される場合、分離された環境で動作しているか

### Dockerfileセキュリティ（Dockerfile存在時）

```dockerfile
# 危険パターン
FROM ubuntu:latest             # latestタグ → 再現性なし、予期しない変更
RUN apt-get install -y ...     # パッケージバージョン未固定
RUN echo $SECRET > /app/.env  # シークレットがレイヤーに残る（docker history で見える）
CMD ["node", "app.js"]         # rootユーザーで実行（デフォルト）

# 推奨
FROM node:20.11-alpine3.19     # バージョン固定 + 最小イメージ
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm install  # シークレットをレイヤーに残さない
RUN addgroup -S app && adduser -S app -G app
USER app                       # 非rootユーザー
HEALTHCHECK --interval=30s CMD curl -f http://localhost:3000/health || exit 1
```

- [ ] ベースイメージがlatestタグでないか（バージョン固定）
- [ ] rootユーザーで実行されていないか（USER命令で非rootに切り替え）
- [ ] ビルドレイヤーにsecret情報が含まれていないか（RUN --mount=type=secret 推奨）
- [ ] .dockerignoreに.env, credentials, .git等が含まれているか
- [ ] マルチステージビルドで最終イメージが最小化されているか（dev依存を含まない）
- [ ] HEALTHCHECKが定義されているか

---

## 視点17: ライセンスコンプライアンス

### ライセンスの確認

```bash
# 依存関係のライセンス一覧
npx license-checker --summary 2>/dev/null | head -30

# プロプライエタリまたは問題のあるライセンスの検出
npx license-checker --failOn "GPL-2.0;GPL-3.0;AGPL-3.0;LGPL-2.0;LGPL-2.1;LGPL-3.0" 2>/dev/null
```

**商用プロジェクトで注意すべきライセンス**:
- **避けるべき**: GPL-2.0, GPL-3.0, AGPL-3.0（コードの公開義務）
- **確認が必要**: LGPL系（リンク方法による）
- **一般的に問題なし**: MIT, Apache-2.0, BSD-2-Clause, BSD-3-Clause, ISC

- [ ] 追加された依存関係のライセンスがプロジェクトに適合しているか
- [ ] GPLなどのコピーレフトライセンスを持つ依存関係がないか
- [ ] ライセンスファイルが存在するか（OSSプロジェクトの場合）

---

## 視点18: スケーラビリティ

### 水平スケーリングへの対応

```typescript
// 危険: ローカルステートへの依存（スケール時に問題）
const sessions = new Map(); // インスタンスごとにリセット

// 危険: グローバル状態
let globalCounter = 0; // 複数インスタンス間で同期されない

// 推奨: 外部ストア（Redis, DB）を使用
const session = await redis.get(`session:${sessionId}`);
```

- [ ] インスタンスをまたぐ状態管理が外部ストレージ（DB/Redis等）に依存しているか
- [ ] ファイルシステムへの書き込みが複数インスタンス環境で問題を起こさないか
- [ ] 設定値がハードコードではなく環境変数から取得されているか

### 負荷増大への対応

- [ ] 重い処理にレート制限/キューイングが実装されているか
- [ ] 外部API呼び出しにリトライとバックオフが実装されているか
- [ ] データ量が増加した場合にインデックスが機能するか（EXPLAIN プランの確認）
- [ ] ページネーションが適切に実装されているか（全件取得を避けているか）

---

## 視点19: 後方互換性

### 破壊的変更の検出

```typescript
// 破壊的変更の例:

// 1. 公開APIのエンドポイント削除
// Before: DELETE /api/v1/orders/:id
// After: 削除 → クライアントが壊れる

// 2. レスポンス形式の変更
// Before: { "data": { "id": 1, "name": "item" } }
// After: { "item": { "id": 1, "name": "item" } } → フィールド名変更

// 3. 必須パラメータの追加
// Before: createUser(email: string)
// After: createUser(email: string, role: string) // role必須追加 → 呼び出し元が壊れる

// 4. 型の変更（TypeScriptライブラリ）
// Before: getId(): number
// After: getId(): string // 型が変わる → コンパイルエラー
```

- [ ] 公開されているAPIエンドポイントが削除・変更されていないか
- [ ] レスポンスの既存フィールドが削除・型変更されていないか
- [ ] 関数のシグネチャが後方互換を維持しているか
- [ ] 破壊的変更がある場合、バージョンを分けているか（/v2/）
- [ ] データベースマイグレーションが既存データに影響しないか

### 非推奨化の適切な管理

```typescript
// 推奨: 段階的な非推奨化
/** @deprecated Use `getUserById()` instead. Will be removed in v3.0 */
function getUser(id: string) { return getUserById(id); }
```

- [ ] 削除された機能が適切に非推奨化プロセスを経ているか
- [ ] CHANGELOGが更新されているか
- [ ] 移行ガイドが提供されているか（ライブラリの場合）

---

## 視点C補足: 並行処理・非同期

### 並行処理の安全性

```typescript
// 危険: 非アトミックな操作（TOCTOU: Time-of-check/time-of-use）
const user = await db.findUser(id);
if (user.credits > 0) { // チェック
  await db.deductCredit(id); // 使用 (この間に他のリクエストが介入できる)
}
// → DB のトランザクション + 楽観的ロックで解決

// 危険: 並列実行の制御なし
const results = await Promise.all(items.map(item => processItem(item)));
// itemsが1000件あると1000並列実行 → DBコネクション枯渇
// → Promise.allSettled + concurrencyLimit で制御

// 危険: 非同期ループでの順序依存
for (const item of items) {
  await processItem(item); // 逐次実行は遅い
}
// 独立した処理なら Promise.all で並列化を検討
```

- [ ] 共有状態の更新にアトミック操作（トランザクション/楽観的ロック）が使われているか
- [ ] 並列実行の同時接続数が制限されているか（DBコネクション枯渇防止）
- [ ] デッドロックの可能性がないか（複数リソースを複数順序でロック）
- [ ] レースコンディションが発生しうる箇所がないか
- [ ] 非同期処理のキャンセル（AbortController等）が考慮されているか
