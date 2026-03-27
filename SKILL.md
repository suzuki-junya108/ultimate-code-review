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
  Do NOT trigger when: the user asks to write new code, refactor code
  (without review intent), explain code, debug a specific error, run tests,
  generate documentation, or asks a general programming question.
---

# Ultimate Code Review Skill

どのプロジェクトでも使用可能なユニバーサル最強コードレビュースキル。
25視点の並列分析 + 6つのメタ分析（他ツールにはない3次元飛躍）を実行する。

---

## 3つの実行モード

| モード | 用途 | 所要時間 | 使用場面 |
|--------|------|---------|---------|
| **full** | 全25視点 + 全6メタ分析 | 15-20分 | PR前、重要機能 |
| **focus** | 指定カテゴリのみ | 5-8分 | 特定の懸念がある時 |
| **quick** | CRITICAL/HIGH検出のみ | 3-5分 | 軽微な変更 |

引数なし → full モード
`/ultimate-code-review focus security` → セキュリティ視点集中
`/ultimate-code-review quick` → 高速スキャン

---

## 実行前チェックリスト

- [ ] 変更ファイルのスコープは明確か（git diff / ステージング済みファイル）
- [ ] プロジェクトルートで実行しているか（モノレポは適切なサブディレクトリ）
- [ ] レビューモードは適切か（full / focus / quick）
- [ ] 対象コードはビルドが通る状態か（構文エラーがあると分析精度が低下）

---

## 5フェーズ実行フロー

```
┌─────────────────────────────────────────────────────────────┐
│ Phase 0: プロジェクト知性の獲得（1-2分）                      │
│  → references/project-intelligence.md を参照               │
└─────────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ Phase 1: 並列25視点分析（4エージェント同時起動）              │
│  Agent-A: カテゴリA 視点1-6   → references/perspective-A.md │
│  Agent-B: カテゴリB 視点7-13  → references/perspective-B.md │
│  Agent-C: カテゴリC 視点14-19 → references/perspective-C.md │
│  Agent-D: カテゴリD 視点20-25 → references/perspective-D.md │
└─────────────────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────────────────┐
│ Phase 2: 6つの横断メタ分析                                    │
│  → references/meta-analysis.md を参照                       │
│  2-1: 視点間相関検出                                          │
│  2-2: 赤チーム分析（攻撃・障害シナリオ）                        │
│  2-3: 進化適応分析（gitログ → フラジリティスコア）              │
│  2-4: 不作為コスト（直さないとどうなるか）                      │
│  2-5: ビジネス価値検証（削除テスト・価値確認）                  │
│  2-6: 変更容易性測定（変更増幅係数）                           │
│  2-7: 認知負荷評価（メンタルモデル複雑さ）                      │
│  2-8: CRITICAL/HIGH問題の自動修正 + 再検証                    │
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

## Phase 1: 25視点並列分析

### 4エージェント同時起動

**実行方法**: 並列エージェント実行が可能な場合は4エージェントを同時起動する。並列実行が利用できない環境では A→B→C→D の順で逐次実行し、結果を統合する。

```
Agent-A（カテゴリA: ユーザー価値）:
  視点1: 機能的正確性
  視点2: セキュリティ
  視点3: パフォーマンス
  視点4: アクセシビリティ
  視点5: 国際化（i18n）
  視点6: エラーハンドリング

Agent-B（カテゴリB: 開発者体験）:
  視点7: アーキテクチャ適合性
  視点8: コード品質
  視点9: テスタビリティ
  視点10: 依存関係管理
  視点11: ドキュメンテーション
  視点12: 保守性
  視点13: 型設計

Agent-C（カテゴリC: 運用・成長）:
  視点14: 技術的負債
  視点15: 可観測性
  視点16: CI/CD統合
  視点17: ライセンスコンプライアンス
  視点18: スケーラビリティ
  視点19: 後方互換性

Agent-D（カテゴリD: 深層技術品質）:
  視点20: 複雑度分析
  視点21: データモデル設計
  視点22: 設定管理
  視点23: 並行処理・非同期
  視点24: API設計原則
  視点25: オンボーディング品質
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
## カテゴリ[A/B/C/D]分析結果

### 視点[N]: [視点名]
評価: ★☆☆☆☆ ~ ★★★★★
[CRITICAL/HIGH/MEDIUM/LOW] [問題の説明] - [ファイル:行番号]
  根拠: [具体的なコード/パターン]
  修正案: [具体的な修正方法]

良い点: [ポジティブな観察]
```

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

---

## リファレンスファイル

| ファイル | 役割 |
|---------|------|
| `references/project-intelligence.md` | Phase 0: タイプ判定・重み付けルール |
| `references/perspective-A.md` | 視点1-6の詳細チェックリスト |
| `references/perspective-B.md` | 視点7-13の詳細チェックリスト |
| `references/perspective-C.md` | 視点14-19の詳細チェックリスト |
| `references/perspective-D.md` | 視点20-25の詳細チェックリスト |
| `references/meta-analysis.md` | 6つのメタ分析ガイド |
| `references/auto-fix-playbook.md` | 自動修正ルールブック |
| `references/executive-synthesis.md` | レポート生成ガイド |
