# Shell / Bash 言語固有ディープレビュー

**品質基準**: Google Shell Style Guide + ShellCheck 全警告 + POSIX sh 仕様委員レベル。
ShellCheck をパスしても残る設計・ロジック問題を検出する。

---

## カテゴリ1: 安全なスクリプト基礎（必須設定）

### CRITICAL

```bash
# NG: 安全オプションなし
#!/bin/bash

do_something
if_it_fails_still_continues  # エラーがあっても処理が続く

# OK: set -euo pipefail を先頭に
#!/usr/bin/env bash
set -euo pipefail

# または個別に設定
set -e   # コマンド失敗で即時終了
set -u   # 未定義変数を参照するとエラー
set -o pipefail  # パイプラインの途中でのエラーを検出
```

```bash
# NG: 変数を引用符なしで展開（word splitting と glob expansion が発生）
filename="my file with spaces.txt"
cat $filename    # NG: "my", "file", "with", "spaces.txt" として解釈される
ls $pattern      # NG: glob が展開される

# OK: 常に二重引用符で囲む
cat "$filename"
ls "$pattern"
```

### HIGH

```bash
# NG: $@ と $* の混同
function process() {
    for arg in $*; do  # NG: スペースを含む引数が分割される
        handle "$arg"
    done
}

# OK: $@ を使う
function process() {
    for arg in "$@"; do  # OK: 各引数を保持
        handle "$arg"
    done
}
```

```bash
# NG: コマンド置換の引用符なし
files=$(ls *.txt)  # ファイル名にスペースがある場合に分割される
process $files     # word splitting が発生

# OK: 引用符で囲む
files=$(ls *.txt)
process "$files"
# またはループで処理
while IFS= read -r file; do
    process "$file"
done < <(find . -name "*.txt")
```

```bash
# NG: [ ] 比較で変数が引用符なし
if [ $status = "active" ]; then  # $status が空だと文法エラー
    ...
fi

# OK: 引用符で囲む
if [ "$status" = "active" ]; then
    ...
fi
# または [[ ]] を使う（bash のみ）
if [[ $status == "active" ]]; then
    ...
fi
```

### MEDIUM

```bash
# NG: local キーワードなし関数内変数（グローバルスコープを汚染）
function setup() {
    temp_dir=/tmp/work$$  # グローバル変数になる
    mkdir -p $temp_dir
}

# OK: local を使う
function setup() {
    local temp_dir
    temp_dir=$(mktemp -d)
    mkdir -p "$temp_dir"
}
```

---

## カテゴリ2: セキュリティ

### CRITICAL

```bash
# NG: eval でユーザー入力を実行
user_input="rm -rf /"
eval "$user_input"  # 任意コード実行

# OK: eval は使わない設計にする
```

```bash
# NG: /tmp に予測可能なファイル名で作成（TOCTOU 競合）
tmpfile=/tmp/script$$
echo "data" > $tmpfile  # $$ は予測可能、symlink 攻撃が可能

# OK: mktemp を使う
tmpfile=$(mktemp)
tmpdir=$(mktemp -d)
echo "data" > "$tmpfile"
```

```bash
# NG: スクリプト内にハードコードされた認証情報
PASSWORD="super_secret_password"
DB_URL="postgresql://user:password@host/db"

# OK: 環境変数から取得
PASSWORD="${DB_PASSWORD:?DB_PASSWORD must be set}"
DB_URL="${DATABASE_URL:?DATABASE_URL must be set}"
```

```bash
# NG: curl | bash（スクリプトの内容を検証せずに実行）
curl https://example.com/install.sh | bash

# ただしこれは一般的な慣習なので、レビュー観点としては:
# - スクリプトのハッシュを検証しているか？
# - HTTPS が使われているか？
# - 信頼できるソースからのみか？
```

### HIGH

```bash
# NG: chmod 777 の多用
chmod 777 /app/scripts  # 全ユーザーに読み書き実行を許可

# OK: 最小権限を付与
chmod 755 /app/scripts  # オーナーのみ書き込み可
chmod 644 config.txt    # 設定ファイルは実行不要
```

---

## カテゴリ3: 堅牢性

### HIGH

```bash
# NG: パイプの終了コードを確認しない
output=$(failing_command | grep "pattern")
# failing_command が失敗しても $? は grep の終了コードになる

# OK: set -o pipefail または PIPESTATUS を確認
set -o pipefail  # パイプの任意コマンドが失敗したら即時終了

# または個別に確認
failing_command | grep "pattern"
exit_codes=("${PIPESTATUS[@]}")
if [[ ${exit_codes[0]} -ne 0 ]]; then
    echo "failing_command failed" >&2
    exit 1
fi
```

```bash
# NG: rm -rf に空の可能性がある変数（rm -rf / になる）
cleanup() {
    rm -rf "$BUILD_DIR"/*  # BUILD_DIR が空だと rm -rf /*
}

# OK: 変数の値を確認
cleanup() {
    if [[ -z "$BUILD_DIR" ]]; then
        echo "BUILD_DIR is not set" >&2
        exit 1
    fi
    rm -rf "${BUILD_DIR:?BUILD_DIR must not be empty}"/*
}
```

```bash
# NG: cd が失敗した場合に後続のコマンドが意図しないディレクトリで実行される
cd /some/dir
rm -rf *  # cd が失敗した場合、カレントディレクトリで rm -rf * が実行される！

# OK: && でチェーンするか、エラー時に終了
cd /some/dir && rm -rf *
# または
cd /some/dir || { echo "Failed to cd" >&2; exit 1; }
rm -rf *
```

### MEDIUM

```bash
# NG: エラー時のロールバック・クリーンアップがない
deploy() {
    backup_current
    deploy_new_version
    # deploy_new_version が失敗した場合、バックアップが復元されない
}

# OK: trap でクリーンアップを設定
deploy() {
    local deployed=false
    trap 'if ! $deployed; then restore_backup; fi' ERR EXIT

    backup_current
    deploy_new_version
    deployed=true
}
```

---

## カテゴリ4: 移植性（POSIX sh vs bash）

### HIGH

```bash
# NG: #!/bin/sh シバンなのに bash 固有拡張を使用
#!/bin/sh

# bash 固有: [[ ]] (POSIX sh では [ ] を使う)
if [[ -n "$var" ]]; then ...

# bash 固有: 配列
arr=(a b c)

# bash 固有: (( )) 算術演算
(( count++ ))

# bash 固有: $RANDOM
echo $RANDOM

# OK: #!/bin/bash に変更するか POSIX sh 互換の書き方にする
#!/bin/sh  # または #!/usr/bin/env bash

# POSIX sh 互換
if [ -n "$var" ]; then ...  # [ ] を使う
count=$((count + 1))         # $(( )) は POSIX 互換
```

```bash
# NG: #!/bin/bash が /usr/bin/bash にあるシステムで動かない
#!/bin/bash  # /usr/bin/bash の場合に失敗

# OK: env を使う（PATH から bash を探す）
#!/usr/bin/env bash
```

```bash
# NG: echo -e（POSIX sh では動作が未定義）
echo -e "line1\nline2"

# OK: printf を使う
printf "line1\nline2\n"
```

### MEDIUM

```bash
# NG: $(...) と `` の混在（読みにくく、ネスト不可）
result=`command1 \`command2\``  # バックティックはネストが複雑

# OK: $(...) に統一（ネスト可能、読みやすい）
result=$(command1 "$(command2)")
```

---

## カテゴリ5: 入出力・リダイレクト

### HIGH

```bash
# NG: read without -r flag（バックスラッシュが処理される）
read line  # "foo\nbar" を読むと "foonbar" になる

# OK: read -r
read -r line  # バックスラッシュをリテラルとして扱う
```

```bash
# NG: IFS を変更して関数を呼んだ後に IFS を戻さない
old_IFS="$IFS"
IFS=","
read -r -a fields <<< "$csv_line"
IFS="$old_IFS"  # 忘れやすい、エラー時に戻されない

# OK: ローカルスコープで変更
parse_csv() {
    local IFS=","  # 関数内のみ有効
    read -r -a fields <<< "$1"
}
```

---

## カテゴリ6: シグナル・プロセス管理

### HIGH

```bash
# NG: trap を設定しないスクリプトで一時ファイルを作成
tmpfile=$(mktemp)
do_work "$tmpfile"
rm -f "$tmpfile"  # Ctrl+C で中断されると tmpfile が残る

# OK: trap で確実にクリーンアップ
tmpfile=$(mktemp)
trap 'rm -f "$tmpfile"' EXIT  # スクリプト終了時（正常・異常問わず）に削除
do_work "$tmpfile"
```

```bash
# NG: バックグラウンドプロセスを起動して wait しない（ゾンビプロセス）
process_file "file1" &
process_file "file2" &
# wait なし → 親プロセスが終了してもバックグラウンドが続く or ゾンビ化

# OK: wait で完了を確認
pids=()
process_file "file1" & pids+=($!)
process_file "file2" & pids+=($!)
for pid in "${pids[@]}"; do
    wait "$pid" || echo "Process $pid failed" >&2
done
```

```bash
# NG: kill -9 を最初から使用（クリーンアップの機会を与えない）
kill -9 "$service_pid"  # SIGKILL: クリーンアップできない

# OK: 段階的にシグナルを送る
kill -TERM "$service_pid"  # まず SIGTERM
sleep 5
if kill -0 "$service_pid" 2>/dev/null; then
    kill -KILL "$service_pid"  # まだ生きていれば SIGKILL
fi
```

---

## カテゴリ7: スタイル・保守性

### HIGH

```bash
# NG: 関数なしで 200 行超のスクリプト
# 全ての処理がトップレベルに書かれており、再利用・テスト不可

# OK: 関数に分割
main() {
    parse_arguments "$@"
    validate_environment
    setup_workspace
    run_deployment
    cleanup
}

parse_arguments() { ... }
validate_environment() { ... }
setup_workspace() { ... }
run_deployment() { ... }
cleanup() { ... }

main "$@"
```

```bash
# NG: ハードコードされたパス
LOG_DIR="/home/ubuntu/myapp/logs"
CONFIG_FILE="/home/ubuntu/myapp/config.yaml"

# OK: 変数化・相対パス・$HOME 使用
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LOG_DIR="${LOG_DIR:-$SCRIPT_DIR/../logs}"
CONFIG_FILE="${CONFIG_FILE:-$SCRIPT_DIR/../config.yaml}"
```

```bash
# NG: 環境変数を無防備に展開
TARGET_DIR=$DEPLOY_TARGET  # DEPLOY_TARGET が未設定だと空文字になる
rm -rf "$TARGET_DIR/*"     # rm -rf /* になる危険性

# OK: デフォルト値か必須チェック
TARGET_DIR="${DEPLOY_TARGET:?DEPLOY_TARGET must be set}"  # 未設定ならエラー終了
# または
TARGET_DIR="${DEPLOY_TARGET:-/tmp/deploy}"  # デフォルト値
```

```bash
# NG: コマンドの終了コードを直後でチェックせず後続処理が継続
cp "$source" "$dest"
# cp が失敗しても（set -e なし）次の行が実行される
process_file "$dest"

# OK: 明示的にチェック
if ! cp "$source" "$dest"; then
    echo "Failed to copy $source to $dest" >&2
    exit 1
fi
process_file "$dest"
```

### MEDIUM

```bash
# NG: 関数名にキャメルケース（shell 慣習は snake_case）
function processUserData() { ... }  # JavaScript 風
function ProcessUserData() { ... }  # Pascal 風

# OK: snake_case を使う
function process_user_data() { ... }
```

```bash
# NG: 定数にクォートがない
readonly MAX_RETRIES=3
readonly LOG_LEVEL=info  # スペースや特殊文字があると問題になる

# OK: 文字列定数は引用符で囲む
readonly MAX_RETRIES=3
readonly LOG_LEVEL="info"
readonly LOG_FORMAT="[%Y-%m-%d %H:%M:%S]"
```
