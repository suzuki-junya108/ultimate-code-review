# Rust 言語固有ディープレビュー

**品質基準**: Rust コンパイラチーム・Rustonomicon・Rust API Guidelines 準拠。
`clippy --deny warnings` をパスしても残る設計・安全性問題を検出する。

---

## カテゴリ1: 所有権・借用の慣用的使用

### CRITICAL

```rust
// NG: unsafe ブロックに SAFETY コメントなし
unsafe {
    let ptr = get_raw_ptr();
    *ptr = 42; // なぜ安全か不明
}

// OK: SAFETY コメントで根拠を説明
// SAFETY: ptr は get_raw_ptr() から取得した有効な非 null ポインタで、
//         この関数の呼び出し中は他のスレッドからアクセスされない
unsafe {
    let ptr = get_raw_ptr();
    *ptr = 42;
}
```

```rust
// NG: std::mem::transmute で安全なキャストを回避
let x: u32 = 42;
let y: i32 = unsafe { std::mem::transmute(x) }; // TryFrom で安全に変換できる

// OK
let y: i32 = i32::try_from(x).map_err(|_| Error::Overflow)?;
```

```rust
// NG: ライブラリコードで unwrap/expect（呼び出し元が panic をコントロールできない）
pub fn parse_config(s: &str) -> Config {
    serde_json::from_str(s).unwrap() // ライブラリ利用者が panic をハンドリングできない
}

// OK: Result を返す
pub fn parse_config(s: &str) -> Result<Config, serde_json::Error> {
    serde_json::from_str(s)
}
```

### HIGH

```rust
// NG: 不要な clone（借用で解決できる）
fn greet(name: String) { // String を所有権ごと受け取る
    println!("Hello, {}", name);
}
greet(user.name.clone()); // 不要な clone

// OK: &str を受け取る
fn greet(name: &str) {
    println!("Hello, {}", name);
}
greet(&user.name);
```

```rust
// NG: as キャストで黙示的切り捨て
let x: u64 = 300;
let y = x as u8; // 44 に切り捨て（サイレントバグ）

// OK: try_from でエラーを明示
let y = u8::try_from(x).map_err(|_| Error::Overflow)?;
```

---

## カテゴリ2: エラーハンドリング設計

### CRITICAL

```rust
// NG: ライブラリ関数で panic!（recoverable error は Result で返す）
pub fn divide(a: i32, b: i32) -> i32 {
    if b == 0 { panic!("division by zero") } // ライブラリの panic はひどいUX
    a / b
}

// OK
pub fn divide(a: i32, b: i32) -> Result<i32, MathError> {
    if b == 0 { return Err(MathError::DivisionByZero) }
    Ok(a / b)
}
```

### HIGH

```rust
// NG: ライブラリで Box<dyn Error>（呼び出し側がエラー型を識別できない）
pub fn process() -> Result<(), Box<dyn Error>> { ... }

// OK: thiserror でカスタムエラー型
#[derive(Debug, thiserror::Error)]
pub enum ProcessError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    #[error("parse error: {0}")]
    Parse(#[from] serde_json::Error),
}
pub fn process() -> Result<(), ProcessError> { ... }
```

```rust
// NG: アプリケーションコードで anyhow::Context なしの ?
let data = fs::read_to_string(path)?; // どのパスで失敗したか不明

// OK: Context でエラーにコンテキスト追加
use anyhow::Context;
let data = fs::read_to_string(path)
    .with_context(|| format!("failed to read {}", path.display()))?;
```

---

## カテゴリ3: イテレータ・コレクション

### HIGH

```rust
// NG: collect してから即ループ（不要なアロケーション）
let doubled: Vec<i32> = nums.iter().map(|x| x * 2).collect();
for v in &doubled { process(v); } // collect が不要

// OK: iterator chain を続ける
nums.iter().map(|x| x * 2).for_each(|v| process(&v));
```

```rust
// NG: インデックスアクセス（バウンドチェックが毎回発生）
for i in 0..vec.len() {
    process(vec[i]);
}

// OK: イテレータ（最適化もしやすい）
for item in &vec {
    process(item);
}
```

```rust
// NG: Option を返す map + flatten が冗長
let result: Vec<Option<String>> = items.iter().map(|x| x.name.clone()).collect();
let names: Vec<String> = result.into_iter().flatten().collect();

// OK: filter_map
let names: Vec<String> = items.iter().filter_map(|x| x.name.clone()).collect();
```

---

## カテゴリ4: トレイト設計

### CRITICAL

```rust
// NG: object-safe でないトレイトに dyn Trait を使用
trait Cloneable {
    fn clone_box(&self) -> Box<dyn Cloneable>;
}
// Self を返すトレイトは dyn では使えない → compile error

// OK: object-safe なトレイトを設計するか、enum dispatch を使う
```

### HIGH

```rust
// NG: 公開型に #[derive(Debug)] がない（デバッグが著しく困難）
pub struct Config {
    pub host: String,
    pub port: u16,
    // Debug なし → println!("{:?}", config) がコンパイルエラー
}

// OK
#[derive(Debug, Clone)]
pub struct Config { ... }
```

```rust
// NG: PartialEq を手動実装して Eq を実装しない（reflexivity が保証されない）
impl PartialEq for MyStruct {
    fn eq(&self, other: &Self) -> bool { ... }
}
// Eq なし → HashMap のキーに使えない（実際の型は a == a が成り立つのに）

// OK: Eq も実装
impl Eq for MyStruct {}
```

```rust
// NG: From<T> と Into<T> を両方実装（自動導出と競合）
impl From<u32> for MyType { ... }
impl Into<MyType> for u32 { ... } // From を実装すると Into が自動導出されるため重複

// OK: From のみ実装（Into は自動導出される）
```

```rust
// NG: Result/Option を返す関数に #[must_use] がない
pub fn save_data(data: &Data) -> Result<(), Error> { ... }
// 呼び出し側が save_data(data); と書いてもコンパイラが警告しない

// OK
#[must_use]
pub fn save_data(data: &Data) -> Result<(), Error> { ... }
```

---

## カテゴリ5: Async Rust

### CRITICAL

```rust
// NG: async fn 内でブロッキング呼び出し（tokio の event loop をブロック）
async fn process_file(path: &str) -> Result<String, Error> {
    let content = std::fs::read_to_string(path)?; // ブロッキング！
    Ok(content)
}

// OK: tokio の非同期 API を使う
async fn process_file(path: &str) -> Result<String, Error> {
    let content = tokio::fs::read_to_string(path).await?;
    Ok(content)
}

// または重い同期処理は spawn_blocking
tokio::task::spawn_blocking(|| std::fs::read_to_string(path)).await?
```

```rust
// NG: std::sync::Mutex を await をまたいで保持（デッドロック）
async fn update(state: Arc<Mutex<State>>) {
    let guard = state.lock().unwrap();
    some_async_operation().await; // await をまたいで Mutex を保持 → デッドロック
    guard.field = new_value;
}

// OK: tokio::sync::Mutex を使う
async fn update(state: Arc<tokio::sync::Mutex<State>>) {
    let mut guard = state.lock().await;
    some_async_operation().await; // tokio Mutex は await をまたいでも安全
    guard.field = new_value;
}
```

### HIGH

```rust
// NG: tokio::spawn で CPU バウンドの重い処理
tokio::spawn(async { heavy_computation() }); // tokio の async runtime を詰まらせる

// OK: spawn_blocking か rayon
tokio::task::spawn_blocking(|| heavy_computation()).await?;
```

```rust
// NG: unbounded channel で無制限バックプレッシャー
let (tx, mut rx) = tokio::sync::mpsc::unbounded_channel::<Item>();
// プロデューサーが速すぎるとメモリが無限に増加

// OK: bounded channel でバックプレッシャーを設ける
let (tx, mut rx) = tokio::sync::mpsc::channel::<Item>(100);
```

---

## カテゴリ6: メモリパターン

### HIGH

```rust
// NG: ホットパスで毎回 String アロケーション
fn get_label(kind: Kind) -> String {
    match kind {
        Kind::Foo => "foo".to_string(), // 毎回ヒープアロケーション
        Kind::Bar => "bar".to_string(),
    }
}

// OK: &'static str を返す
fn get_label(kind: Kind) -> &'static str {
    match kind {
        Kind::Foo => "foo",
        Kind::Bar => "bar",
    }
}
```

```rust
// NG: サイズ既知のコレクションに with_capacity を使わない
let mut v = Vec::new();
for item in items { v.push(item); } // 複数回 realloc

// OK
let mut v = Vec::with_capacity(items.len());
for item in items { v.push(item); }
```

```rust
// NG: 1 MB 超のスタックアロケーション（スタックオーバーフローリスク）
fn process() {
    let buffer = [0u8; 2 * 1024 * 1024]; // 2MB スタックに積む
    ...
}

// OK: ヒープに移動
fn process() {
    let buffer = vec![0u8; 2 * 1024 * 1024];
    ...
}
```

---

## カテゴリ7: unsafe コード監査（全 unsafe ブロックを精査）

### CRITICAL

```rust
// NG: NULL チェックなしの raw pointer dereference
unsafe {
    let ptr: *const u8 = get_ptr();
    let val = *ptr; // ptr が null の場合 UB
}

// OK
unsafe {
    let ptr: *const u8 = get_ptr();
    if ptr.is_null() { return Err(Error::NullPointer); }
    let val = *ptr;
}
```

```rust
// NG: 未信頼ソースから len を受け取る slice::from_raw_parts
unsafe fn from_ffi(ptr: *const u8, len: usize) -> &'static [u8] {
    std::slice::from_raw_parts(ptr, len) // len が ptr の実際のサイズを超える可能性
}
// バッファオーバーリード → UB
```

```rust
// NG: ライフタイム延長（dangling reference につながる）
fn extend_lifetime<'a>(r: &'a str) -> &'static str {
    unsafe { std::mem::transmute::<&'a str, &'static str>(r) }
    // r が drop された後に使われると UB
}
```

### HIGH

```rust
// NG: unsafe impl Send/Sync に根拠コメントなし
unsafe impl Send for MyWrapper {}
// なぜ Send が安全か説明なし

// OK
// SAFETY: MyWrapper は内部に Arc<Mutex<T>> を保持しており、
//         複数スレッドからの同時アクセスは Mutex で保護されている
unsafe impl Send for MyWrapper {}
```

---

## カテゴリ8: Cargo・エコシステム

### HIGH

```toml
# NG: edition が古い（新規プロジェクト）
[package]
edition = "2018"  # または指定なし

# OK
[package]
edition = "2021"
```

```toml
# NG: binary crate の Cargo.lock を .gitignore（再現性の喪失）
# .gitignore に Cargo.lock が含まれている
# → library crate は .gitignore でよいが、binary は必ずコミット

# OK: binary crate では Cargo.lock をコミット
```

```toml
# NG: library crate に MSRV (Minimum Supported Rust Version) の記載なし
[package]
name = "my-lib"
# rust-version なし → ユーザーが必要バージョンを把握できない

# OK
[package]
rust-version = "1.70"
```

```toml
# NG: デフォルト feature が重い依存を引き込む
[dependencies]
serde = "1"  # デフォルトで "derive" feature が入る
tokio = "1"  # デフォルトで全 feature が入る

# OK: 必要な feature のみ
[dependencies]
serde = { version = "1", default-features = false, features = ["derive"] }
tokio = { version = "1", features = ["rt", "macros"] }
```

---

## カテゴリ9: Clippy 相当の深層パターン

`clippy::pedantic` および `clippy::restriction` lint に相当するパターンを手動確認する。

| lint | チェック内容 |
|------|------------|
| `clippy::unwrap_used` | lib コードの全 `.unwrap()` を検出 |
| `clippy::expect_used` | lib コードの全 `.expect()` を検出 |
| `clippy::redundant_clone` | 不要な `.clone()` を検出 |
| `clippy::map_unwrap_or` | `.map().unwrap_or()` → `.map_or()` に統一 |
| `clippy::if_then_some_else_none` | `bool::then()` / `bool::then_some()` を使う |
| `clippy::manual_let_else` | `let ... else` 構文 (1.65+) を活用 |
| `clippy::large_enum_variant` | enum variant のサイズ不均衡 → `Box<LargeType>` |
| `clippy::wildcard_imports` | `use foo::*` の禁止（テスト以外） |
| `clippy::exhaustive_structs` | 公開型に `#[non_exhaustive]` を付けるべきケース |

```rust
// NG: map + unwrap_or
let val = opt.map(|x| x * 2).unwrap_or(0);

// OK
let val = opt.map_or(0, |x| x * 2);
```

```rust
// NG: if-else で Some/None を手動構築
let val = if condition { Some(x) } else { None };

// OK
let val = condition.then(|| x);
// または
let val = condition.then_some(x);
```

---

## カテゴリ10: 高度なパターン・型システム活用

### HIGH

```rust
// NG: 意味的に異なる ID 型を同じプリミティブ型で表現
fn transfer(from: u64, to: u64, amount: u64) { ... }
transfer(user_id, order_id, amount); // 引数の入れ替えをコンパイラが検出できない

// OK: newtype パターン
struct UserId(u64);
struct OrderId(u64);
struct Amount(u64);
fn transfer(from: UserId, to: UserId, amount: Amount) { ... }
```

```rust
// NG: Option<Option<T>> の使用（意味が不明確）
fn find_user(id: Option<UserId>) -> Option<Option<User>> { ... }
// None = ID なし? User なし? 区別不可

// OK: 専用 enum を定義
enum LookupResult {
    NoId,
    NotFound,
    Found(User),
}
```

---

## カテゴリ11: FFI 安全性

### CRITICAL

```rust
// NG: extern "C" 関数が Rust の panic を C 側に伝播させる（UB）
#[no_mangle]
pub extern "C" fn process_data(ptr: *const u8, len: usize) -> i32 {
    let slice = unsafe { std::slice::from_raw_parts(ptr, len) };
    do_work(slice) // panic が C 境界を越える → UB
}

// OK: catch_unwind でガード
#[no_mangle]
pub extern "C" fn process_data(ptr: *const u8, len: usize) -> i32 {
    let result = std::panic::catch_unwind(|| {
        let slice = unsafe { std::slice::from_raw_parts(ptr, len) };
        do_work(slice)
    });
    result.unwrap_or(-1) // panic をエラーコードに変換
}
```

### HIGH

```rust
// NG: C 側から受け取った文字列を unchecked で変換
extern "C" fn process_str(ptr: *const c_char) {
    let s = unsafe { std::str::from_utf8_unchecked(/* ... */) }; // UTF-8 検証なし
}

// OK: 検証付きで変換
extern "C" fn process_str(ptr: *const c_char) {
    let c_str = unsafe { std::ffi::CStr::from_ptr(ptr) };
    let s = c_str.to_str().map_err(|_| Error::InvalidUtf8)?;
}
```

---

## カテゴリ12: WebAssembly 固有（wasm-bindgen 検出時）

### HIGH

```rust
// NG: console_error_panic_hook が設定されておらず panic がサイレント
#[wasm_bindgen(start)]
pub fn main() {
    // console_error_panic_hook::set_once() がない
    // → panic しても browser console に何も出ない
}

// OK
#[wasm_bindgen(start)]
pub fn main() {
    console_error_panic_hook::set_once();
}
```

```rust
// NG: JsValue エラーを無視
#[wasm_bindgen]
pub fn process(data: &[u8]) {
    let _ = do_work(data); // JsValue エラーを握りつぶす
}

// OK: エラーを返す
#[wasm_bindgen]
pub fn process(data: &[u8]) -> Result<(), JsValue> {
    do_work(data).map_err(|e| JsValue::from_str(&e.to_string()))
}
```

### MEDIUM

```rust
// NG: 大きな Rust オブジェクトを wasm_bindgen で JS 側に公開（コピーコストが高い）
#[wasm_bindgen]
pub fn get_large_data() -> Vec<u8> { // 毎回 WASM → JS へコピー
    large_computation()
}

// OK: JS 側から直接 WASM メモリにアクセスするか、ポインタを返す設計
```
