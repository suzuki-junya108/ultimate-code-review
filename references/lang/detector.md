# 言語検出・バージョン判定プロトコル

Phase 0 のステップ2.5で実行される。主要言語を検出し、対応する lang ファイルへのパスと
バージョン固有の有効チェックマトリクスを出力する。

---

## 言語検出カスケード

以下の優先順位で検出する（先にマッチしたものが主要言語）:

```bash
# 検出コマンド例
ls Cargo.toml go.mod pyproject.toml setup.py requirements.txt \
   pom.xml build.gradle build.gradle.kts Package.swift Gemfile \
   composer.json tsconfig.json package.json 2>/dev/null

# .kt ファイルの存在確認（Kotlin 判定）
find . -name "*.kt" -not -path "*/node_modules/*" -not -path "*/.git/*" | head -1

# シェルスクリプトの確認
find . -name "*.sh" -o -name "*.bash" | head -1
```

| 優先順位 | 検出条件 | 主要言語 | lang ファイル |
|---------|---------|---------|-------------|
| 1 | `Cargo.toml` 存在 | Rust | `references/lang/rust.md` |
| 2 | `go.mod` 存在 | Go | `references/lang/go.md` |
| 3 | `pyproject.toml` / `setup.py` / `requirements.txt` 存在 | Python | `references/lang/python.md` |
| 4 | `pom.xml` or `build.gradle.kts` + `.kt` ファイル存在 | Kotlin | `references/lang/kotlin.md` |
| 5 | `pom.xml` / `build.gradle` 存在 | Java | `references/lang/java.md` |
| 6 | `Package.swift` / `*.xcodeproj` / `*.xcworkspace` 存在 | Swift | `references/lang/swift.md` |
| 7 | `Gemfile` / `.ruby-version` 存在、または `.rb` ファイルが大半 | Ruby | `references/lang/ruby.md` |
| 8 | `composer.json` 存在、または `.php` ファイルが大半 | PHP | `references/lang/php.md` |
| 9 | `*.sh` / `*.bash` ファイル存在、またはシバン `#!/bin/bash` / `#!/bin/sh` | Shell | `references/lang/shell.md` |
| 10 | `tsconfig.json` 存在 | TypeScript | `references/lang/typescript.md` |
| 11 | `package.json` のみ | JavaScript | `references/lang/typescript.md`（JS 節を読む） |
| — | 上記いずれも非該当 | 未対応 | スキップ |

---

## バージョン検出ルール

### Python
```
優先順位:
1. .python-version ファイル → 内容がバージョン文字列
2. pyproject.toml: requires-python = ">=3.11" → 3.11
3. pyproject.toml: [tool.poetry.dependencies] python = "^3.10" → 3.10
4. Dockerfile: FROM python:3.12-slim → 3.12
5. フォールバック: 3.8 と仮定（最も広くサポートされる最小）
```

### Go
```
go.mod ファイルの先頭行:
  go 1.21 → バージョン 1.21
  go 1.18 → バージョン 1.18
フォールバック: 1.16 と仮定
```

### Rust
```
優先順位:
1. rust-toolchain.toml: channel = "stable" / "1.75.0"
2. Cargo.toml: edition = "2021" / "2018" / "2015"
フォールバック: edition 2018 と仮定
```

### Java
```
優先順位:
1. pom.xml: <java.version>21</java.version>
2. pom.xml: <maven.compiler.source>17</maven.compiler.source>
3. build.gradle: sourceCompatibility = '17'
4. build.gradle.kts: jvmTarget = "17"
フォールバック: Java 11 と仮定
```

### PHP
```
composer.json: "require": { "php": ">=8.1" } → 8.1
フォールバック: PHP 7.4 と仮定
```

### Ruby
```
優先順位:
1. .ruby-version ファイル
2. Gemfile: ruby '3.2'
フォールバック: Ruby 2.7 と仮定
```

---

## バージョン別チェック有効化マトリクス

### Python

| チェック項目 | 必要バージョン | それ未満の場合 |
|------------|-------------|-------------|
| walrus operator `(x := expr)` チェック | >= 3.8 | N/A とマーク |
| `X \| Y` union 型ヒント（`__future__` なし） | >= 3.10 | 別チェック内容を適用 |
| Structural pattern matching (`match`) | >= 3.10 | N/A |
| `asyncio.timeout()` | >= 3.11 | `asyncio.wait_for()` を推奨 |
| `ExceptionGroup` / `TaskGroup` | >= 3.11 | N/A |
| `dict[str, int]` (PEP 585) 組み込み generics | >= 3.9 | `Dict[str, int]` が必要 |
| `X \| None` (PEP 604) | >= 3.10 | `Optional[X]` が必要 |

### Go

| チェック項目 | 必要バージョン | それ未満の場合 |
|------------|-------------|-------------|
| Generics (`constraints.Ordered` 等) | >= 1.18 | N/A |
| `any` キーワード（`interface{}` の alias） | >= 1.18 | `interface{}` が正しい表記 |
| `slices` / `maps` stdlib パッケージ | >= 1.21 | 再実装を咎めない |
| `errors.Join` | >= 1.20 | N/A |
| ループ変数キャプチャ修正 | >= 1.22 | **チェック有効**: `go func() { use(v) }()` バグを検出 |
| `log/slog` | >= 1.21 | N/A |

### Rust

| チェック項目 | 必要バージョン | それ未満の場合 |
|------------|-------------|-------------|
| `let ... else` 構文 | >= 1.65 (edition 2021) | N/A |
| `async fn` in trait (AFIT) | >= 1.75 | `async-trait` crate が必要、それなしを HIGH に |
| `std::sync::OnceLock` | >= 1.70 | `once_cell` crate 使用を容認 |
| Edition 2021 クロージャキャプチャ | edition = "2021" | 旧挙動を前提にチェック |

### Java

| チェック項目 | 必要バージョン | それ未満の場合 |
|------------|-------------|-------------|
| Records | >= 16 | N/A |
| Sealed classes | >= 17 | N/A |
| Pattern matching for switch | >= 21 | N/A |
| Virtual threads | >= 21 | N/A |
| `instanceof` パターン変数 | >= 16 | 従来の explicit cast を容認 |
| Text blocks | >= 15 | N/A |
| `.toList()` (immutable) | >= 16 | `Collectors.toList()` を容認 |

---

## マルチ言語モノレポの扱い

```
手順:
1. 変更ファイルの拡張子ごとにカウント
   例: .py: 8, .ts: 3, .sh: 1

2. 最多拡張子の言語 → 主要言語（フル Phase 1.5 実行）

3. 2ファイル以上の副次言語 → CRITICAL/HIGH チェックのみ実行

4. Phase 0 出力に記録:
   **主要言語**: Python 3.11（8ファイル）
   **副次言語**: TypeScript（3ファイル, CRITICAL/HIGHのみ）, Shell（1ファイル, スキップ）
```

### よくあるマルチ言語組み合わせ

| 組み合わせ | 典型的なプロジェクト |
|-----------|-------------------|
| TypeScript + Python | フロントエンド + バックエンド API |
| Go + TypeScript | API サーバー + フロントエンド |
| Rust + TypeScript | Tauri / ネイティブ拡張 + フロントエンド |
| Kotlin + Shell | Android アプリ + ビルドスクリプト |
| PHP + JavaScript | WordPress / Laravel + フロントエンド |
| Ruby + JavaScript | Rails + Stimulus/Turbo |

---

## 未対応言語のグレースフルデグレード

対応 lang ファイルが存在しない言語を検出した場合:

```
1. Phase 0 出力に記録:
   [LANG:UNSUPPORTED: C#]
   言語固有レビュー: 未実施（references/lang/csharp.md が存在しない）

2. Phase 1.5 をスキップ

3. Phase 1 の各エージェントに通知:
   「lang ファイルなし: C#。各自の知識で言語固有リスクを評価すること」

4. レポートに記載:
   ## 🔤 言語固有レビュー結果
   ⚠️ C# は未対応言語です。言語固有の深層チェックは実施されていません。
   Phase 1 の25視点分析でカバーされる範囲のみ評価されています。
```

### 現在未対応の言語

C#, C, C++, Haskell, Scala, Clojure, Elixir, Erlang, Dart/Flutter, R, MATLAB, Perl, Lua

---

## Phase 0 出力フォーマット（言語検出フィールド追加分）

```markdown
**主要言語**: Python 3.11（pyproject.toml: requires-python = ">=3.11"）
**言語固有レビューファイル**: references/lang/python.md
**副次言語**: Shell（2ファイル, CRITICAL/HIGHのみ）
**バージョン固有の無効チェック**: なし（3.11は全チェック有効）
```
