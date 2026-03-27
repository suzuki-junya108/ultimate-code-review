# Ultimate Code Review — Claude Code Skill

**29視点の並列分析 + 9つのメタ分析**による、Claude Code向け最強コードレビュースキル。

OWASP/CWE/NIST/Google SRE Book/GDPR/CCPA/CIS Benchmark等、世界中のセキュリティ標準・SREベストプラクティスを網羅。他のツールにはない「3次元飛躍」（赤チーム分析・ビジネス価値検証・認知負荷評価）で、単なる構文チェックを超えた本質的なコード品質を評価します。

---

## 特徴

| 機能 | 説明 |
|------|------|
| **29視点並列分析** | 5エージェントが同時起動し、ユーザー価値・開発者体験・運用・深層品質・横断品質を網羅 |
| **OWASP Web + API Top 10完全網羅** | CSRF, SSRF, Mass Assignment, ReDoS, DOM XSS, JWT攻撃, API特有脆弱性を含む全20項目 |
| **SOLID原則完全版** | SRPだけでなくOCP/LSP/ISP/DIPも含む全5原則を検証 |
| **テスト品質分析** | テストピラミッド比率・Contract Testing・Mutation Testing（Stryker）を評価 |
| **OpenTelemetry/Prometheus品質** | メトリクス命名規則・Label Cardinality爆発・分散トレーシング整合性を検証 |
| **サプライチェーンセキュリティ** | Dependency Confusion・lockファイル整合性・シークレットgit履歴スキャンを実施 |
| **GDPR/CCPA実装確認** | 削除権（hard delete）・データポータビリティ・PII検出・Cookie同意を検証 |
| **SRE信頼性工学** | Circuit Breaker・タイムアウト階層・冪等性・SLO/エラーバジェットを評価 |
| **インフラ/コンテナセキュリティ** | Dockerfile CIS Benchmark・GitHub Actions SHA pinning・Kubernetes PSS を確認 |
| **赤チーム分析** | 攻撃者・障害担当者の視点でシステムの弱点を特定 |
| **SLOインパクト評価** | デプロイ戦略（即時/カナリア/フィーチャーフラグ）をリスクベースで提言 |
| **言語固有ディープレビュー（10言語）** | Python/Go/Rust/TypeScript/Java/Kotlin/Swift/PHP/Ruby/Shellの専門チェックをPhase 1.5で実行 |
| **自動修正（30パターン）** | CRITICAL/HIGH問題を自動修正しlint/typecheck/test/buildで検証 |
| **無限スケール対応** | 変更ファイルが30件超でも品質を落とさず全ファイルを完全レビュー（チャンク完全分析モード） |
| **モノレポ対応** | nx/turbo/pnpm-workspace/go.work/Cargo workspaceを自動検出し関連ファイルをまとめて分析 |

---

## インストール

このスキルは [Claude Code](https://claude.ai/code) の **カスタムスキル** として動作します。

### セットアップ手順

1. このリポジトリをクローン:
   ```bash
   git clone https://github.com/suzuki-junya108/ultimate-code-review.git
   ```

2. `SKILL.md` と `references/` フォルダをあなたのプロジェクトの `.claude/skills/ultimate-code-review/` にコピー:
   ```bash
   mkdir -p your-project/.claude/skills/ultimate-code-review
   cp -r ultimate-code-review/. your-project/.claude/skills/ultimate-code-review/
   ```

3. Claude Code の `settings.json` にスキルを登録:
   ```json
   {
     "skills": [
       {
         "name": "ultimate-code-review",
         "path": ".claude/skills/ultimate-code-review/SKILL.md"
       }
     ]
   }
   ```

4. Claude Code を再起動すると `/ultimate-code-review` コマンドが使えるようになります。

---

## 使い方

```
# フルレビュー（29視点 + Phase 1.5言語固有 + 9メタ分析）
/ultimate-code-review

# 高速スキャン（CRITICAL/HIGH検出のみ）
/ultimate-code-review quick

# 特定カテゴリに集中
/ultimate-code-review focus security
/ultimate-code-review focus performance
/ultimate-code-review focus architecture

# スコープを絞る（任意）
/review --path src/auth              # 特定ディレクトリのみ
/review --package api                # モノレポの特定パッケージのみ
/review --base main                  # mainとの差分を対象に
/review --staged                     # ステージング済みファイルのみ
/review focus security --path src/auth  # 組み合わせも可
```

または会話の中で自然に:
- 「最強レビューして」
- 「包括的レビューお願い」
- 「ultimate review」

**大規模PRも安心**: 変更ファイルが30件を超えると自動的にチャンク完全分析モードに移行し、品質を落とさず全ファイルをレビューします。

---

## 実行フロー

```
Phase 0: プロジェクト知性の獲得（スケール検出・モノレポ検出・プロジェクトタイプ判定）
    ↓
Phase 1: 29視点完全分析（変更ファイル ≤ 30: 一括実行 / > 30: チャンク完全分析）
    Agent-A: カテゴリA（機能的正確性・セキュリティ全網羅・パフォーマンス・アクセシビリティ・i18n・エラーハンドリング）
    Agent-B: カテゴリB（アーキテクチャ/SOLID完全版・コード品質・テスタビリティ/テストピラミッド・依存関係・ドキュメント・保守性・型設計）
    Agent-C: カテゴリC（技術的負債・可観測性/OpenTelemetry・CI/CD/GitHub Actions/Dockerfile・ライセンス・スケーラビリティ・後方互換性）
    Agent-D: カテゴリD（複雑度・データモデル・設定管理・並行処理・API設計/GraphQL/WebSocket・オンボーディング・サプライチェーン）
    Agent-E: カテゴリE（プライバシー/GDPR・SRE/信頼性工学・インフラ/コンテナセキュリティ）
    ↓
Phase 1.5: 言語固有ディープレビュー
    Python / Go / Rust / TypeScript / Java / Kotlin / Swift / PHP / Ruby / Shell
    （主要言語: 全カテゴリ実行 / 副次言語: CRITICAL/HIGH のみ）
    ↓
Phase 2: 9つのメタ分析
    2-1: 視点間相関検出（根本原因の特定）
    2-2: 赤チーム分析（攻撃・障害シナリオ）
    2-3: 進化適応分析（フラジリティスコア）
    2-4: 不作為コスト（放置した場合の影響）
    2-5: ビジネス価値検証（削除テスト・仕様乖離）
    2-6: 変更容易性測定（変更増幅係数）
    2-7: 認知負荷評価（Miller's Law適用）
    2-8: CRITICAL/HIGH問題の自動修正 + 再検証
    2-9: SLOインパクト評価（デプロイリスク・推奨戦略）
    ↓
Phase 3: エグゼクティブレポート生成
```

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

## レポートサンプル

```markdown
# Ultimate Code Review 結果

**総合評価**: B（CRITICALなし、HIGH 2件）

| 重要度 | 件数 | 自動修正済み |
|--------|------|------------|
| CRITICAL | 0 | - |
| HIGH | 2 | 1 |
| MEDIUM | 8 | - |
| LOW | 5 | - |

**技術的負債トレンド**: +3件/90日 ⚠️増加中

## 🔴 赤チーム分析
### 最危険攻撃経路
APIキーがログに出力される可能性 - src/api/client.ts:42

## 🔄 変更容易性
注文ステータス変更の変更増幅係数: **8ファイル** 🔴（目標: ≤3）

## 🧠 認知負荷評価
processOrder() - 前提知識数: 9（上限7を超過）→ 分割推奨
```

---

## ファイル構成

```
.
├── SKILL.md                          # メインスキル定義
├── README.md                         # このファイル
└── references/
    ├── scale-strategy.md             # Phase 0 Step 0: スケール検出・チャンク完全分析・モノレポ検出
    ├── project-intelligence.md       # Phase 0: タイプ判定・重み付けルール
    ├── perspective-A.md              # 視点1-6（ユーザー価値・OWASP全網羅）
    ├── perspective-B.md              # 視点7-13（開発者体験・SOLID完全版・テストピラミッド）
    ├── perspective-C.md              # 視点14-19（運用・成長・GitHub Actions・Dockerfile）
    ├── perspective-D.md              # 視点20-26（深層技術品質・GraphQL/WebSocket・サプライチェーン）
    ├── perspective-E.md              # 視点27-29（GDPR・SRE信頼性工学・インフラセキュリティ）
    ├── meta-analysis.md              # 9つのメタ分析ガイド（SLOインパクト評価含む）
    ├── auto-fix-playbook.md          # 自動修正ルールブック（30パターン）
    ├── executive-synthesis.md        # レポート生成ガイド
    └── lang/
        ├── detector.md               # Phase 1.5: 言語検出・バージョン判定プロトコル
        ├── python.md                 # Python専門チェック（12カテゴリ）
        ├── go.md                     # Go専門チェック（13カテゴリ）
        ├── rust.md                   # Rust専門チェック（12カテゴリ）
        ├── typescript.md             # TypeScript/JS専門チェック（8カテゴリ）
        ├── java.md                   # Java専門チェック（7カテゴリ）
        ├── kotlin.md                 # Kotlin専門チェック（7カテゴリ）
        ├── swift.md                  # Swift専門チェック（7カテゴリ）
        ├── php.md                    # PHP専門チェック（8カテゴリ）
        ├── ruby.md                   # Ruby専門チェック（7カテゴリ）
        └── shell.md                  # Shell/Bash専門チェック（7カテゴリ）
```

---

## 動作要件

- [Claude Code](https://claude.ai/code) CLI または IDE拡張
- カスタムスキル機能のサポート（Claude Code 最新版）

---

## ライセンス

MIT License

---

## 作者

[@suzuki-junya108](https://github.com/suzuki-junya108)
