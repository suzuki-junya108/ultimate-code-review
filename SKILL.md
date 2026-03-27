---
name: ultimate-code-review
license: MIT
description: |
  Run a comprehensive, multi-perspective code review on any codebase.
  Trigger this skill when the user asks for: code review, コードレビュー,
  ultimate review, 最強レビュー, 包括的レビュー, comprehensive review,
  deep review, full review, PR review, pull request review, security review,
  performance review, architecture review, 品質チェック, 品質レビュー,
  コード品質, review this PR, review my code, review these changes,
  check my code, analyze this code, コードを見て, レビューして.
  Also trigger for: /ultimate-code-review, /review, /code-review.
  Also trigger for: /review --path, /review --package, /review --base, /review --staged.
  Do NOT trigger when: the user asks to write new code, refactor code
  (without review intent), explain code, debug a specific error, run tests,
  generate documentation, or asks a general programming question.
---

# Ultimate Code Review Skill

どのプロジェクトでも使用可能なユニバーサル最強コードレビュースキル。
29視点の並列分析 + 9つのメタ分析（他ツールにはない3次元飛躍）を実行する。

---

## 3つの実行モード

| モード | 用途 | 所要時間 | 使用場面 |
|--------|------|---------|---------|
| **full** | 全29視点 + 全9メタ分析 | 25-35分（1チャンク分） | PR前、重要機能 |
| **focus** | 指定カテゴリのみ | 5-8分 | 特定の懸念がある時 |
| **quick** | CRITICAL/HIGH検出のみ | 3-5分 | 軽微な変更 |

引数なし → full モード
`/ultimate-code-review focus security` → セキュリティ視点集中
`/ultimate-code-review quick` → 高速スキャン

**スケール自動対応**: 変更ファイルが30件を超えると、チャンク完全分析モードに自動移行。
品質は一切落とさず、全ファイルを29視点で完全レビュー（所要時間は増加）。
詳細は `references/scale-strategy.md` を参照。

---

## スコープオプション（任意）

デフォルトは `git diff HEAD~1 HEAD` の全変更ファイルが対象。以下のオプションでスコープを絞れる:

| オプション | 説明 | 例 |
|-----------|------|-----|
| `--path <dir>` | 指定パス以下のみ対象 | `/review --path src/auth` |
| `--package <name>` | モノレポの特定パッケージのみ | `/review --package api` |
| `--base <ref>` | 差分ベースリビジョンを指定 | `/review --base main` |
| `--staged` | ステージング済みファイルのみ | `/review --staged` |

オプションは組み合わせ可能: `/review focus security --path src/auth`

**注**: スコープを絞らなくても、大規模PRは自動的にチャンク完全分析で対応する。

---

## 実行前チェックリスト

- [ ] 変更ファイルのスコープは明確か（git diff / ステージング済みファイル）
- [ ] プロジェクトルートで実行しているか
- [ ] レビューモードは適切か（full / focus / quick）
- [ ] 対象コードはビルドが通る状態か（構文エラーがあると分析精度が低下）
- [ ] 変更ファイルが多い場合: `--path` や `--package` でスコープを絞れば時間を短縮できる（任意）

---

## 5フェーズ実行フロー

```
┌─────────────────────────────────────────────────────────────┐
│ Phase 0: プロジェクト知性の獲得（1-2分）                      │
│  Step 0: スケール検出 & チャンク計画                          │
│  → references/scale-strategy.md を参照                     │
│  → references/project-intelligence.md を参照               │
└─────────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ Phase 1: 29視点完全分析（チャンク対応）                        │
│  変更ファイル ≤ 30: 5エージェント一括実行（従来通り）            │
│  変更ファイル > 30: チャンク別に完全分析 × N回（20件/チャンク）  │
│  Agent-A: カテゴリA 視点1-6   → references/perspective-A.md │
│  Agent-B: カテゴリB 視点7-13  → references/perspective-B.md │
│  Agent-C: カテゴリC 視点14-19 → references/perspective-C.md │
│  Agent-D: カテゴリD 視点20-26 → references/perspective-D.md │
│  Agent-E: カテゴリE 視点27-29 → references/perspective-E.md │
└─────────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ Phase 1.5: 言語固有ディープレビュー（3-5分）                   │
│  言語検出: references/lang/detector.md                       │
│  Python  → references/lang/python.md                        │
│  Go      → references/lang/go.md                            │
│  Rust    → references/lang/rust.md                          │
│  TS/JS   → references/lang/typescript.md                    │
│  Java    → references/lang/java.md                          │
│  Kotlin  → references/lang/kotlin.md                        │
│  Swift   → references/lang/swift.md                         │
│  PHP     → references/lang/php.md                           │
│  Ruby    → references/lang/ruby.md                          │
│  Shell   → references/lang/shell.md                         │
│  未対応  → スキップ（[LANG:UNSUPPORTED]記録）                 │
└─────────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ Phase 2: 9つの横断メタ分析                                    │
│  → references/meta-analysis.md を参照                       │
│  2-1: 視点間相関検出                                          │
│  2-2: 赤チーム分析（攻撃・障害シナリオ）                        │
│  2-3: 進化適応分析（gitログ → フラジリティスコア）              │
│  2-4: 不作為コスト（直さないとどうなるか）                      │
│  2-5: ビジネス価値検証（削除テスト・価値確認）                  │
│  2-6: 変更容易性測定（変更増幅係数）                           │
│  2-7: 認知負荷評価（メンタルモデル複雑さ）                      │
│  2-8: CRITICAL/HIGH問題の自動修正 + 再検証                    │
│  2-9: SLOインパクト評価（デプロイリスク・戦略）                 │
└─────────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ Phase 3: エグゼクティブレポート生成                            │
│  → references/executive-synthesis.md を参照                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Phase 0: プロジェクト知性の獲得

### 実行手順

**ステップ0: スケール検出 & チャンク計画**（`references/scale-strategy.md` を参照）

0-a. **引数解析**: `--path` / `--package` / `--base` / `--staged` を抽出する
0-b. **変更ファイルリスト取得**: git diff でスコープファイルを取得し、フィルタを適用する
0-c. **スケール判定**: ファイル数をカウントし SCALE_CONTEXT を生成する
0-d. **モノレポ検出**: `nx.json` / `turbo.json` / `pnpm-workspace.yaml` / `go.work` / `Cargo.toml[workspace]` を確認する
0-e. **チャンク計画**: ファイル数 > 30 の場合、20件ごとのチャンクに分割する（関連ファイルを優先的に同一チャンクにまとめる）
0-f. **所要時間告知**: チャンク数 × 推定時間をユーザーに提示する
   例: `45ファイル検出 → 3チャンク → 推定 65-90分`

---

1. **ドキュメント読み込み**
   - `README.md` / `CLAUDE.md` → ビジネス文脈・要件
   - `docs/design/` → 設計書（あれば）
   - `.claude/patterns.md` / `.claude/decisions.md` → プロジェクト固有ルール（あれば）

2. **プロジェクトタイプ判定**
   ```
   package.json / 主要ファイルを確認:
   - next.js / react → Webフロントエンド
   - express / hono / fastify / workers → REST API
   - *.ts exports only → ライブラリ/パッケージ
   - db schemas / ORMs → データ集約型
   - auth / payment mentions → 認証・課金系
   - commander / yargs → CLI/ツール
   ```

3. **開発フェーズ判定**
   - git log件数 < 50 → PoC/プロトタイプ
   - テストファイルが少ない → MVP
   - CI/CD + 本番環境あり → 本番運用
   - リファクタリングPR → リファクタリングフェーズ

4. **変更対象の特定**
   ```bash
   git diff --name-only HEAD~1 HEAD 2>/dev/null || git diff --name-only --cached
   ```
   変更ファイルのリストを取得し、レビュー対象を明確化

5. **重み付け決定**（`references/project-intelligence.md` 参照）

---

## Phase 1: 29視点完全分析

### 実行方法（スケールに応じて自動選択）

**変更ファイル ≤ 30（通常モード）**: 並列エージェント実行が可能な場合は5エージェントを同時起動する。並列実行が利用できない環境では A→B→C→D→E の順で逐次実行し、結果を統合する。

**変更ファイル > 30（チャンク完全分析モード）**: `references/scale-strategy.md` のチャンク完全分析プロトコルに従う。各チャンクに対して下記と同じ5エージェント×29視点の完全分析を実行し、全チャンクの findings を結合してから Phase 1.5 へ進む。

進捗をユーザーに表示する:
```
[チャンク 1/3 処理中] ファイル1〜20 ...
[チャンク 2/3 処理中] ファイル21〜40 ...
[チャンク 3/3 処理中] ファイル41〜45 ...
[全チャンク完了] findings 結合中 → Phase 1.5 へ
```

### 5エージェント同時起動

```
Agent-A（カテゴリA: ユーザー価値）:
  視点1: 機能的正確性
  視点2: セキュリティ（OWASP Web/API Top 10, CSRF, SSRF, JWT攻撃, ReDoS等）
  視点3: パフォーマンス
  視点4: アクセシビリティ
  視点5: 国際化（i18n）
  視点6: エラーハンドリング

Agent-B（カテゴリB: 開発者体験）:
  視点7: アーキテクチャ適合性（SOLID完全版）
  視点8: コード品質
  視点9: テスタビリティ（テストピラミッド・Contract Testing・Mutation Testing）
  視点10: 依存関係管理
  視点11: ドキュメンテーション
  視点12: 保守性
  視点13: 型設計

Agent-C（カテゴリC: 運用・成長）:
  視点14: 技術的負債
  視点15: 可観測性（OpenTelemetry・Prometheusメトリクス品質・Label Cardinality）
  視点16: CI/CD統合（GitHub Actions セキュリティ・Dockerfileセキュリティ）
  視点17: ライセンスコンプライアンス
  視点18: スケーラビリティ
  視点19: 後方互換性

Agent-D（カテゴリD: 深層技術品質）:
  視点20: 複雑度分析
  視点21: データモデル設計
  視点22: 設定管理
  視点23: 並行処理・非同期
  視点24: API設計原則（GraphQL/WebSocket固有セキュリティ含む）
  視点25: オンボーディング品質
  視点26: サプライチェーンセキュリティ（Dependency Confusion・シークレットスキャン）

Agent-E（カテゴリE: 横断品質）:
  視点27: プライバシー・コンプライアンス（GDPR/CCPA・PII管理）
  視点28: SRE・信頼性工学（Circuit Breaker・タイムアウト階層・冪等性・SLO）
  視点29: インフラ・コンテナセキュリティ（Dockerfile CIS・GitHub Actions・K8s PSS）
```

### 各エージェントへの指示テンプレート

```
あなたはコードレビュー専門エージェントです。
以下の視点でコードを分析し、構造化された結果を返してください。

対象ファイル: [変更ファイルのリスト]
プロジェクトタイプ: [Phase 0で判定したタイプ]
フェーズ: [Phase 0で判定したフェーズ]

分析する視点: [カテゴリ名と視点番号リスト]
詳細ガイド: [該当するperspective-*.mdの内容]

出力形式:
## カテゴリ[A/B/C/D/E]分析結果

### 視点[N]: [視点名]
評価: ★☆☆☆☆ ~ ★★★★★
[CRITICAL/HIGH/MEDIUM/LOW] [問題の説明] - [ファイル:行番号]
  根拠: [具体的なコード/パターン]
  修正案: [具体的な修正方法]

良い点: [ポジティブな観察]
```

---

## Phase 1.5: 言語固有ディープレビュー

### 実行条件

| モード | 動作 |
|--------|------|
| **full** | 主要言語: 全カテゴリ実行 / 副次言語（2ファイル以上）: CRITICAL/HIGH のみ |
| **focus** | 言語固有チェックのみ実行（focus lang 指定時）または通常フルスキャン |
| **quick** | CRITICAL/HIGH のみ実行（全カテゴリ対象） |

### 実行手順

1. `references/lang/detector.md` を読み込み、主要言語・バージョンを検出
2. 対応する `references/lang/[language].md` を読み込む
3. バージョン固有の無効チェックを `detector.md` のマトリクスで確認し N/A をマーク
4. 変更ファイルを対象に各カテゴリのチェックを実行

### エージェントへの指示テンプレート

```
あなたは[言語名]専門コードレビューエージェントです。
[言語名] [バージョン] の言語固有チェックを実行してください。

対象ファイル: [変更ファイルのリスト]
言語固有ガイド: references/lang/[language].md
バージョン固有の無効チェック: [detector.mdから取得したリスト]

CRITICAL/HIGH を優先して報告し、根拠となるコードスニペットと修正案を含めてください。
```

### Phase 1.5 結果の扱い

- 言語固有 CRITICAL は Phase 2 の相関分析 (2-1) で Phase 1 の結果と統合する
- Phase 1 でも同じ問題が指摘された場合: severity を1段階昇格（HIGH → CRITICAL）
- レポートの `📋 詳細レビュー結果` に `🔤 言語固有レビュー結果` セクションを追加する

---

## Phase 2: メタ分析

`references/meta-analysis.md` の詳細ガイドに従って実行:

### 2-1: 視点間相関検出
Phase 1の結果を横断して同一根本原因を探す。
複数視点で同じ問題が検出された場合、優先度を昇格させる。

### 2-2 〜 2-7: 詳細メタ分析

以下の順で実行する（`references/meta-analysis.md` 参照）:
1. 赤チーム分析（攻撃・障害シナリオ）
2. 進化適応分析（gitログ → フラジリティスコア算出）
3. 不作為コスト（問題を放置した場合の週次・月次・年次影響）
4. ビジネス価値検証（削除テスト・過剰設計検出・仕様乖離）
5. 変更容易性測定（変更増幅係数 = 1変更で何ファイル要修正か）
6. 認知負荷評価（Miller's Law: 前提知識数 > 7 を超過した箇所）

全2-1〜2-7の結果を確認後、CRITICAL/HIGH問題が残る場合のみ2-8を実行する。

### 2-8: 自動修正
`references/auto-fix-playbook.md` 参照

### 2-9: SLOインパクト評価
`references/meta-analysis.md` の2-9を参照。
デプロイリスクとカナリア/フィーチャーフラグ戦略を提言する。

---

## Phase 3: レポート生成

`references/executive-synthesis.md` の出力フォーマットに従ってレポートを生成する。

---

## 評価グレード

| グレード | 意味 | 基準 |
|---------|------|------|
| **A** | 優秀 | CRITICAL/HIGH問題なし、全視点★4以上 |
| **B** | 良好 | CRITICALなし、HIGH 1-3件 |
| **C** | 改善必要 | HIGH 4-8件 または CRITICAL 1件 |
| **D** | 大幅改善必要 | CRITICAL 2件以上 または HIGH 9件以上 |
| **E** | 重大問題 | システムの安全性・正確性に関わる問題 |

---

## 自動修正フロー

```
CRITICAL/HIGH 問題検出
    ↓
auto-fix-playbook.md でパターンマッチング
    ↓
修正可能 → 自動修正 → lint/typecheck/test/build で検証
修正不可 → [MANUAL REQUIRED] タグ付きでレポートに追加
    ↓
再スキャン（修正済み問題を除外）
```

---

## 使用例

### 例1: デフォルト（fullモード）

**入力:**
```
/ultimate-code-review
```

**出力例（サマリー部分）:**
```
## エグゼクティブサマリー
総合評価: B（CRITICALなし、HIGH 2件）
CRITICAL: 0件  HIGH: 2件（自動修正済み: 1件）  MEDIUM: 5件  LOW: 3件

🔴 HIGH: src/auth/middleware.ts:42 — JWT署名検証が省略されている
  修正案: jwt.verify(token, secret, { algorithms: ["HS256"] })
🔴 HIGH: src/api/users.ts:18 — N+1クエリ（ループ内DB呼び出し）
  修正案: db.user.findMany({ where: { id: { in: ids } } })
```

### 例2: セキュリティ集中モード

**入力:**
```
/ultimate-code-review focus security
```

**出力例:**
```
カテゴリA 視点2（セキュリティ）のみ実行。
CRITICAL: ハードコードされたAPIキー検出 — src/config.ts:5
  const API_KEY = "sk-proj-xxxx" → process.env.API_KEY に変更
```

---

## 失敗パターンと回復手順

### F-1: Phase 0 でプロジェクトタイプを特定できない

**症状**: `package.json` も `Cargo.toml` も見つからず、タイプ判定が「不明」になる。
**原因**: モノレポのサブディレクトリで実行した、または非標準の構成。
**回復手順**:
1. ユーザーに確認する:「どのディレクトリがプロジェクトルートですか？」
2. 手動でタイプを指定して継続:「REST APIとして分析します」
3. `references/project-intelligence.md` の「その他」フォールバックを適用する。

### F-2: Phase 1 でエージェントの一部が結果を返さない

**症状**: 4エージェントのうち1〜2が空の出力を返す、または応答が極端に遅い。
**原因**: 対象ファイルが大きすぎる（1万行超）かタイムアウト。
**回復手順**:
1. 対象ファイルを200行ずつのチャンクに分割して再投入する。
2. タイムアウトした視点番号を特定し、単独で再実行する。
3. `quick` モードに切り替えて CRITICAL/HIGH のみを対象にする。

### F-3: Phase 2 で相関グループが空になる

**症状**: 2-1 視点間相関検出が「相関なし」を返し、メタ分析の価値が出ない。
**原因**: Phase 1 の各エージェントが問題を0件しか検出しなかった。
**回復手順**:
1. ★3以下の視点を再確認し「問題なし」と「検出困難」を区別する。
2. git hotspotを直接分析して潜在的問題を探す。
3. 全視点で健全と判断し Grade A でレポートする。

### F-4: Phase 1.5 で言語が検出できない、または未対応言語

**症状**: `detector.md` の検出ルールにマッチするファイルが存在しない。または検出された言語の `references/lang/` ファイルが存在しない。
**原因**: C#, Scala, Dart 等の未対応言語、または非標準のプロジェクト構成。
**回復手順**:
1. Phase 0 出力に記録: `[LANG:UNSUPPORTED: C#]` / `言語固有レビュー: 未実施`
2. Phase 1.5 をスキップし Phase 2 へ進む
3. Phase 1 の各エージェントに通知: 「lang ファイルなし: C#。各自の知識で言語固有リスクを評価すること」
4. レポートの `🔤 言語固有レビュー結果` セクションに警告を記載:
   `⚠️ [言語名] は未対応言語です。言語固有の深層チェックは実施されていません。`

### F-5: チャンク処理中にエージェントがタイムアウト（チャンク完全分析モードのみ）

**症状**: チャンク N のエージェントが空の出力を返す、または応答が極端に遅い。
**原因**: そのチャンクに含まれるファイルが大きすぎる（1ファイルが1万行超など）。
**回復手順**: `references/scale-strategy.md` の F-5 プロトコルを参照。
1. そのチャンク（20件）をさらに10件×2チャンクに分割して再投入する
2. 成功した場合は findings を蓄積して残りのチャンクを続行する
3. 再度失敗した場合: そのファイルを「手動レビュー推奨」としてレポートに記録し、残りのチャンクは続ける

### F-6: モノレポのパッケージ境界が不明確（チャンク完全分析モードのみ）

**症状**: ファイルがどのパッケージに属するか判定できない（ルートファイル等）。
**回復手順**: `references/scale-strategy.md` の F-6 プロトコルを参照。
1. ルートファイルは「インフラファイル」として分類し通常のディレクトリ順グループ化にフォールバック

### F-7: --path / --package フィルタが 0 件にマッチ

**症状**: 指定したパスに変更ファイルが存在しない。
**回復手順**: `references/scale-strategy.md` の F-7 プロトコルを参照。
1. エラーメッセージを表示し、変更ファイルの候補リストを提示する
2. フィルタなしで続行するかユーザーに確認する

---

## リファレンスファイル

| ファイル | 役割 |
|---------|------|
| `references/scale-strategy.md` | Phase 0 Step 0: スケール検出・チャンク完全分析・モノレポ検出プロトコル |
| `references/project-intelligence.md` | Phase 0: タイプ判定・重み付けルール |
| `references/perspective-A.md` | 視点1-6の詳細チェックリスト（セキュリティ: OWASP全網羅） |
| `references/perspective-B.md` | 視点7-13の詳細チェックリスト（SOLID完全版・テストピラミッド） |
| `references/perspective-C.md` | 視点14-19の詳細チェックリスト（GitHub Actions・Dockerfile） |
| `references/perspective-D.md` | 視点20-26の詳細チェックリスト（GraphQL/WebSocket・サプライチェーン） |
| `references/perspective-E.md` | 視点27-29の詳細チェックリスト（GDPR・SRE・インフラセキュリティ） |
| `references/meta-analysis.md` | 9つのメタ分析ガイド（SLOインパクト評価含む） |
| `references/auto-fix-playbook.md` | 自動修正ルールブック（30パターン） |
| `references/executive-synthesis.md` | レポート生成ガイド |
| `references/lang/detector.md` | Phase 1.5: 言語検出・バージョン判定プロトコル |
| `references/lang/python.md` | Phase 1.5: Python専門チェック（12カテゴリ） |
| `references/lang/go.md` | Phase 1.5: Go専門チェック（13カテゴリ） |
| `references/lang/rust.md` | Phase 1.5: Rust専門チェック（12カテゴリ） |
| `references/lang/typescript.md` | Phase 1.5: TypeScript/JS専門チェック（8カテゴリ） |
| `references/lang/java.md` | Phase 1.5: Java専門チェック（7カテゴリ） |
| `references/lang/kotlin.md` | Phase 1.5: Kotlin専門チェック（7カテゴリ） |
| `references/lang/swift.md` | Phase 1.5: Swift専門チェック（7カテゴリ） |
| `references/lang/php.md` | Phase 1.5: PHP専門チェック（8カテゴリ） |
| `references/lang/ruby.md` | Phase 1.5: Ruby専門チェック（7カテゴリ） |
| `references/lang/shell.md` | Phase 1.5: Shell/Bash専門チェック（7カテゴリ） |
