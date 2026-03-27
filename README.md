# Ultimate Code Review — Claude Code Skill

**25視点の並列分析 + 8つのメタ分析**による、Claude Code向け最強コードレビュースキル。

他のツールにはない「3次元飛躍」（赤チーム分析・ビジネス価値検証・認知負荷評価）で、単なる構文チェックを超えた本質的なコード品質を評価します。

---

## 特徴

| 機能 | 説明 |
|------|------|
| **25視点並列分析** | 4エージェントが同時起動し、ユーザー価値・開発者体験・運用・深層技術品質を網羅 |
| **赤チーム分析** | 攻撃者・障害担当者の視点でシステムの弱点を特定 |
| **進化適応分析** | gitログからフラジリティスコアを算出し「壊れやすい箇所」を定量化 |
| **ビジネス価値検証** | 「このコードを削除したら何が困るか？」で不要なコードを特定 |
| **変更容易性測定** | 変更増幅係数（1つの変更で何ファイルの修正が必要か）を測定 |
| **認知負荷評価** | Miller's Lawを適用し「理解しにくいコード」を定量化 |
| **不作為コスト分析** | 問題を放置した場合の時間軸別影響を予測 |
| **自動修正** | CRITICAL/HIGH問題を自動修正しlint/typecheck/test/buildで検証 |

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
# フルレビュー（全25視点 + 全メタ分析）
/ultimate-code-review

# 高速スキャン（CRITICAL/HIGH検出のみ）
/ultimate-code-review quick

# 特定カテゴリに集中
/ultimate-code-review focus security
/ultimate-code-review focus performance
/ultimate-code-review focus architecture
```

または会話の中で自然に:
- 「最強レビューして」
- 「包括的レビューお願い」
- 「ultimate review」

---

## 実行フロー

```
Phase 0: プロジェクト知性の獲得（プロジェクトタイプ・フェーズ判定）
    ↓
Phase 1: 25視点並列分析（4エージェント同時起動）
    Agent-A: カテゴリA（機能的正確性・セキュリティ・パフォーマンス・アクセシビリティ・i18n・エラーハンドリング）
    Agent-B: カテゴリB（アーキテクチャ・コード品質・テスタビリティ・依存関係・ドキュメント・保守性・型設計）
    Agent-C: カテゴリC（技術的負債・可観測性・CI/CD・ライセンス・スケーラビリティ・後方互換性）
    Agent-D: カテゴリD（複雑度・データモデル・設定管理・並行処理・API設計・オンボーディング）
    ↓
Phase 2: 8つのメタ分析
    2-1: 視点間相関検出（根本原因の特定）
    2-2: 赤チーム分析（攻撃・障害シナリオ）
    2-3: 進化適応分析（フラジリティスコア）
    2-4: 不作為コスト（放置した場合の影響）
    2-5: ビジネス価値検証（削除テスト・仕様乖離）
    2-6: 変更容易性測定（変更増幅係数）
    2-7: 認知負荷評価（Miller's Law適用）
    2-8: CRITICAL/HIGH問題の自動修正 + 再検証
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
    ├── project-intelligence.md       # Phase 0: タイプ判定・重み付けルール
    ├── perspective-A.md              # 視点1-6（ユーザー価値）
    ├── perspective-B.md              # 視点7-13（開発者体験）
    ├── perspective-C.md              # 視点14-19（運用・成長）
    ├── perspective-D.md              # 視点20-25（深層技術品質）
    ├── meta-analysis.md              # 8つのメタ分析ガイド
    ├── auto-fix-playbook.md          # 自動修正ルールブック
    └── executive-synthesis.md        # レポート生成ガイド
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
