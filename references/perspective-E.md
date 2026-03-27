# カテゴリE: 拡張品質（視点27-29）

プライバシー・コンプライアンス・SRE・インフラ・クラウドセキュリティを扱う拡張視点。
これらはセキュリティエンジニア・SREエンジニア・プライバシーエンジニアがレビューする領域をカバーする。

**適用条件**:
- 視点27（プライバシー）: 個人データを扱うシステム（ECサイト、SaaS、認証系）
- 視点28（SRE）: 本番稼働中または本番化予定のシステム
- 視点29（インフラ）: Dockerfile / .github/workflows / k8s マニフェストが存在する場合

---

## 視点27: プライバシー・コンプライアンス

**プロジェクトに個人データの取り扱いがない場合はスキップ可**

### GDPR（一般データ保護規則）の義務確認

**削除権（Right to be Forgotten）**
```typescript
// 危険: soft delete のみ（物理データが残る）
app.delete('/users/:id', async (req, res) => {
  await db.user.update(id, { deletedAt: new Date() }); // ← GDPRでは不十分
});

// 推奨: hard delete + 匿名化された監査ログ
app.delete('/users/:id', async (req, res) => {
  await db.transaction(async (tx) => {
    await tx.userPii.delete({ where: { userId: id } }); // 個人データ物理削除
    await tx.orders.updateMany({ where: { userId: id }, data: { userId: null } }); // 匿名化
    await tx.auditLog.create({ data: { action: 'USER_DELETED', anonymizedId: hash(id) } });
  });
});
```

**チェックリスト:**
- [ ] ユーザーデータの削除が **hard delete** で実装されているか（soft deleteのみはGDPR違反の可能性）
- [ ] hard deleteできない場合、個人識別フィールドが匿名化されているか
- [ ] データポータビリティAPIが存在するか（`GET /users/me/export` 等）
- [ ] ユーザーデータ収集に明示的同意（opt-in, pre-checked box NG）があるか
- [ ] データ保持期間ポリシーが定義・実装されているか
- [ ] 第三者サービスへのデータ転送が最小限か（分析ツール, 広告等）

### PII（個人識別情報）の管理

**コード内のPIIハードコード検出:**
```regex
メールアドレス: [a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}
電話番号（日本）: 0[0-9]{9,10}
クレジットカード: [0-9]{4}[- ]?[0-9]{4}[- ]?[0-9]{4}[- ]?[0-9]{4}
マイナンバー: [0-9]{12}
```

**ログへのPII出力検出:**
```typescript
// 危険
logger.info("User login", { user }); // email, name等が含まれる可能性
logger.error("Auth failed", { email, password }); // 危険

// 安全: PIIフィールドをマスク
logger.info("User login", { userId: user.id, role: user.role });
logger.error("Auth failed", { userId: user?.id }); // emailも不要
```

- [ ] ログにメールアドレス・氏名・電話番号・クレジットカード番号が出力されていないか
- [ ] エラーレスポンスにPIIが含まれていないか
- [ ] データベース上でPIIが暗号化されているか（保存時暗号化）

### Cookie・トラッキング

- [ ] 必須でないCookieに同意なしで設定していないか
- [ ] Cookieに `HttpOnly`, `Secure`, `SameSite` 属性が設定されているか
- [ ] 第三者トラッキングスクリプト（Google Analytics等）に同意取得があるか

---

## 視点28: SRE・信頼性工学

**MVPフェーズでは Circuit Breaker と冪等性のみ確認し、残りは寛容に**

### Circuit Breaker / フォールバック設計

```typescript
// 危険: 外部APIへの直接呼び出し（障害時にリソースが枯渇）
const data = await fetch(externalAPI, { timeout: 30000 }); // フォールバックなし
// 外部APIが503を返し続けると→ 全スレッドが30秒ブロック → サービス停止

// 推奨: Circuit Breaker パターン
import CircuitBreaker from 'opossum';
const breaker = new CircuitBreaker(
  () => fetch(externalAPI, { timeout: 20000 }),
  {
    timeout: 25000,
    errorThresholdPercentage: 50,  // 50%失敗でOPEN
    resetTimeout: 30000,           // 30秒後にHALF-OPEN
    fallback: () => ({ cached: true, data: getLastKnownValue() }), // フォールバック
  }
);
```

- [ ] 外部API呼び出しにサーキットブレーカーがあるか（opossum, resilience4j等）
- [ ] 外部APIが全停止した場合のフォールバック動作が定義されているか
- [ ] 外部API障害がシステム全体に伝播しない設計（バルクヘッド）になっているか

### タイムアウト階層

```typescript
// 危険: 全てが同じタイムアウト
// → どのコンポーネントがタイムアウトしたか不明
const DB_TIMEOUT = 30000;
const SERVER_TIMEOUT = 30000;
const CLIENT_TIMEOUT = 30000;

// 推奨: クライアント > サーバー > DB の階層
const DB_TIMEOUT = 20000;       // DBが最初に諦める
const SERVER_TIMEOUT = 25000;   // サーバーがDB失敗を確認できる
const CLIENT_TIMEOUT = 30000;   // クライアントがサーバー失敗を確認できる
```

- [ ] タイムアウトが「クライアント > サーバー > DB/外部API」の順になっているか
- [ ] 全てのHTTP呼び出し、DB操作、外部API呼び出しにタイムアウトが設定されているか
- [ ] タイムアウトが無制限（0またはundefined）になっていないか

### 冪等性（Idempotency）

```typescript
// 危険: リトライで2回実行される
app.post('/api/payments', async (req, res) => {
  await chargeCard(req.body.cardId, req.body.amount); // ネットワーク断でリトライ→2回請求
});

// 推奨: Idempotency Key で1回だけ実行を保証
app.post('/api/payments', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'];
  if (!idempotencyKey) return res.status(400).json({ error: 'Idempotency-Key required' });

  const existing = await cache.get(`payment:${idempotencyKey}`);
  if (existing) return res.json(JSON.parse(existing)); // 前回の結果を返す

  const result = await chargeCard(req.body.cardId, req.body.amount);
  await cache.set(`payment:${idempotencyKey}`, JSON.stringify(result), 86400); // 24h
  return res.json(result);
});
```

- [ ] 支払い・メール送信・在庫更新等のエンドポイントに冪等性が保証されているか
- [ ] リトライ時に重複実行されない設計か（Idempotency Key, DB unique constraint等）
- [ ] PUT/PATCH は設計上冪等か（同じリクエストを複数回実行して安全か）

### グレースフルシャットダウン

```typescript
// 危険: SIGTERM時に即座に終了（進行中のリクエストが中断される）
process.on('SIGTERM', () => {
  process.exit(1); // 危険: 処理中のリクエストが強制切断
});

// 推奨: 新規リクエストを停止し、既存の処理を完了してから終了
const server = app.listen(PORT);
process.on('SIGTERM', async () => {
  console.log('Graceful shutdown started...');
  server.close(async () => {  // 新規リクエストの受け入れ停止
    await db.disconnect();    // DB接続をクローズ
    await mq.close();         // MQ接続をクローズ
    process.exit(0);
  });
  // K8s のgrace period（デフォルト30秒）内に完了するよう強制終了
  setTimeout(() => { console.error('Forced shutdown'); process.exit(1); }, 25000);
});
```

- [ ] `SIGTERM` シグナルハンドラーが実装されているか
- [ ] 新規リクエスト停止後、進行中の処理が完了してから終了するか
- [ ] K8sのterminationGracePeriodSeconds（デフォルト30秒）内に終了できるか

### SLO/SLA・ヘルスチェック

```typescript
// Liveness vs Readiness の区別
// Liveness: プロセスが生きているか（失敗→再起動）
app.get('/health/live', (req, res) => {
  res.json({ status: 'ok' }); // 常にOK（プロセスが動いていれば）
});

// Readiness: トラフィック受け入れ可能か（失敗→トラフィック一時停止）
app.get('/health/ready', async (req, res) => {
  const dbOk = await db.ping().catch(() => false);
  const redisOk = await redis.ping().catch(() => false);
  if (!dbOk || !redisOk) return res.status(503).json({ status: 'not ready' });
  res.json({ status: 'ok', db: dbOk, redis: redisOk });
});
```

- [ ] ヘルスチェックエンドポイントが存在するか（`/health` または `/health/ready`）
- [ ] Liveness（プロセス生存）とReadiness（依存サービス確認）が分けられているか
- [ ] 重要APIにレイテンシー目標（P99 < Xms）が定義・測定されているか
- [ ] エラーレート目標が定義・モニタリングされているか

---

## 視点29: インフラ・コンテナセキュリティ

**Dockerfile / .github/workflows / k8s マニフェストが存在する場合のみ適用**

### Dockerfile セキュリティ（CIS Docker Benchmark）

```dockerfile
# ❌ 危険パターン集
FROM ubuntu:latest             # latestタグ（再現性なし、意図しない更新）
RUN apt-get install curl       # キャッシュクリアなし（肥大化）
RUN echo $API_KEY > /app/.env  # シークレットがレイヤーに残存
COPY . .                       # .envが含まれる可能性（.dockerignore確認が必要）
CMD ["node", "app.js"]         # rootユーザーで実行（特権コンテナ）

# ✅ 推奨パターン
FROM node:20.11-alpine3.18                                  # 固定バージョン + 軽量イメージ
RUN apk add --no-cache curl                                  # --no-cacheでキャッシュ削減
RUN --mount=type=secret,id=api_key,target=/run/secrets/api_key \ # ビルドシークレット
    cat /run/secrets/api_key > /app/config.txt
COPY --chown=node:node . .                                   # 所有者設定
RUN addgroup -S appgroup && adduser -S appuser -G appgroup   # 専用ユーザー作成
USER appuser                                                 # 非rootユーザー
HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost/health || exit 1
```

**チェックリスト:**
- [ ] `FROM` でlatestタグを使っていないか（固定バージョン指定）
- [ ] `USER` 命令で非rootユーザーに切り替えているか
- [ ] ビルドレイヤーにシークレット情報（APIキー、パスワード）が残っていないか（`docker history` で確認可能）
- [ ] `.dockerignore` に `.env`, `credentials`, `*.pem`, `node_modules` が含まれているか
- [ ] マルチステージビルドで最終イメージが最小化されているか
- [ ] `HEALTHCHECK` 命令が設定されているか

### GitHub Actions セキュリティ

```yaml
# ❌ 危険パターン
on: pull_request_target         # PR内容がプッシュ後コードを実行可能（危険）
jobs:
  build:
    permissions: write-all      # 全権限（不要な権限を付与）
    steps:
      - uses: actions/checkout@v4              # タグ（サプライチェーン攻撃可能）
      - uses: unknown-org/unknown-action@main  # 不明なアクション

# ✅ 推奨パターン
on: pull_request
jobs:
  build:
    permissions:                # 最小権限
      contents: read
      checks: write
    steps:
      # アクションをコミットSHAにピン留め（改ざん防止）
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      - name: Build
        env:
          API_KEY: ${{ secrets.API_KEY }}     # secrets経由で渡す
          # API_KEY: "hardcoded-key"          # ❌ ハードコード禁止
```

**チェックリスト:**
- [ ] サードパーティアクションがコミットSHAにピン留めされているか（`@v4` ではなく `@sha256...`）
- [ ] `GITHUB_TOKEN` の `permissions` が最小権限か（`write-all` は危険）
- [ ] ワークフロー内にシークレットがハードコードされていないか
- [ ] `pull_request_target` イベントの使用に注意しているか（フォークPRが特権コードを実行できる）
- [ ] self-hosted runnerが使われている場合、エフェメラル（使い捨て）設定か

### Kubernetes セキュリティ（k8sマニフェストが存在する場合）

```yaml
# ❌ 危険パターン
spec:
  containers:
  - name: app
    securityContext:
      privileged: true               # フルシステムアクセス
      runAsUser: 0                   # root実行
      allowPrivilegeEscalation: true # 権限昇格可能

# ✅ 推奨パターン
spec:
  containers:
  - name: app
    securityContext:
      runAsNonRoot: true
      runAsUser: 1000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true    # 読み取り専用ファイルシステム
      capabilities:
        drop: [ALL]                   # 全capabilityを削除
        add: [NET_BIND_SERVICE]       # 必要なもののみ追加
    resources:                        # リソース制限
      limits:
        memory: "512Mi"
        cpu: "500m"
      requests:
        memory: "128Mi"
        cpu: "100m"
```

**チェックリスト:**
- [ ] `privileged: true` が使われていないか
- [ ] `runAsUser: 0`（root）で実行されていないか
- [ ] `allowPrivilegeEscalation: false` が設定されているか
- [ ] `resources.limits` が設定されているか（未設定だとノードリソースを枯渇させる）
- [ ] NetworkPolicy が設定されているか（デフォルトは全通信許可）
- [ ] Secretsが YAML に平文で記載されていないか（外部シークレットストアを使用しているか）

### secrets のgit履歴汚染チェック

```bash
# .envファイルが現在のgitで追跡されていないか確認
git ls-files .env .env.local .env.production .env.staging .env.development
# → 出力があれば CRITICAL: gitから削除してgit-filterを実行する必要あり

# git履歴にシークレットが含まれていないか確認（パターン例）
git log --oneline -p -- .env 2>/dev/null | head -50
# → 過去のコミットに.envの変更が含まれていれば要注意
```

- [ ] `.env` ファイルが `.gitignore` に含まれているか
- [ ] `git ls-files .env*` で出力がないか（gitで追跡されていないか）
- [ ] `git log -S 'API_KEY'` で過去コミットにシークレットが含まれていないか
- [ ] `.env.example` に実際の値ではなく、プレースホルダーが記載されているか
