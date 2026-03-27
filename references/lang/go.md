# Go 言語固有ディープレビュー

**品質基準**: Google 社内 Go スタイルガイド + Effective Go + Go メモリモデル（2022年改訂版）準拠。
`golangci-lint` をパスしても残る問題を検出する。

---

## カテゴリ1: Effective Go イディオム

### CRITICAL

```go
// NG: ライブラリコードで panic
func ParseConfig(data []byte) Config {
    if data == nil {
        panic("nil data") // 呼び出し元がハンドリング不可
    }
}

// OK: エラーを返す
func ParseConfig(data []byte) (Config, error) {
    if data == nil {
        return Config{}, errors.New("nil data")
    }
}
```

### HIGH

```go
// NG: エラーを _ で無視
val, _ = strconv.Atoi(s) // s が非数値の場合、val = 0 で正常値と区別不可

// OK: エラーハンドリング
val, err := strconv.Atoi(s)
if err != nil {
    return fmt.Errorf("invalid integer %q: %w", s, err)
}
```

```go
// NG: %w を使わない文字列結合
return fmt.Errorf("parse error: " + err.Error()) // wrapping なし

// OK: %w で wrapping
return fmt.Errorf("parse error: %w", err)
```

```go
// NG: init() で I/O
func init() {
    db, _ = sql.Open("postgres", os.Getenv("DB_URL")) // テスト不可、初期化順序依存
}

// OK: 明示的な初期化関数
func NewApp(dbURL string) (*App, error) {
    db, err := sql.Open("postgres", dbURL)
    ...
}
```

```go
// NG: Go 慣習違反のゲッター名
func (u *User) GetName() string { return u.name }

// OK: Go 慣習
func (u *User) Name() string { return u.name }
```

### MEDIUM

```go
// NG: 複雑な関数での Named return values
func process(data []byte) (result string, err error) {
    defer func() {
        if err != nil {
            result = "" // defer との相互作用が複雑
        }
    }()
    ...
}

// OK: 単純な関数か、エラー時の defer クリーンアップが明確な場合のみ
```

---

## カテゴリ2: エラーハンドリング

### CRITICAL

```go
// NG: defer Close のエラーを無視
func writeFile(path string, data []byte) error {
    f, err := os.Create(path)
    if err != nil { return err }
    defer f.Close() // 書き込みバッファのフラッシュ失敗を検出できない

    _, err = f.Write(data)
    return err
}

// OK: Close のエラーを確認
func writeFile(path string, data []byte) (retErr error) {
    f, err := os.Create(path)
    if err != nil { return err }
    defer func() {
        if err := f.Close(); err != nil && retErr == nil {
            retErr = err
        }
    }()
    _, retErr = f.Write(data)
    return
}
```

### HIGH

```go
// NG: == でセンチネルエラー比較（wrapping で壊れる）
if err == io.EOF { ... }

// OK: errors.Is で比較
if errors.Is(err, io.EOF) { ... }
```

```go
// NG: 型アサーション（wrapping で失敗）
if e, ok := err.(*os.PathError); ok { ... }

// OK: errors.As
var pathErr *os.PathError
if errors.As(err, &pathErr) { ... }
```

```go
// NG: エラーメッセージが大文字で始まる
return errors.New("File not found") // ログ連結時に "error: File not found" になる

// OK: 小文字で始める、句読点なし
return errors.New("file not found")
```

```go
// NG: エラーが最後でない戻り値
func getUser() (error, *User) { ... } // 慣習違反

// OK: エラーは最後
func getUser() (*User, error) { ... }
```

---

## カテゴリ3: Goroutine 管理

### CRITICAL

```go
// NG: 終了機構なし goroutine（goroutine leak）
go func() {
    for {
        process()
        time.Sleep(time.Second)
    }
}()

// OK: context によるキャンセル
go func(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            process()
            time.Sleep(time.Second)
        }
    }
}(ctx)
```

```go
// NG: WaitGroup.Add を goroutine 内で呼ぶ（Add 前に Wait が通過するレース）
for _, item := range items {
    go func(item Item) {
        wg.Add(1) // NG: goroutine 起動後に Add
        defer wg.Done()
        process(item)
    }(item)
}
wg.Wait()

// OK: goroutine 起動前に Add
for _, item := range items {
    wg.Add(1)
    go func(item Item) {
        defer wg.Done()
        process(item)
    }(item)
}
wg.Wait()
```

### HIGH

```go
// NG: Go 1.22 未満でのループ変数キャプチャ（全て最終値になる）
for _, v := range items {
    go func() {
        fmt.Println(v) // Go 1.22 未満: 全て最後の v の値
    }()
}

// OK: 引数として渡す
for _, v := range items {
    go func(v Item) {
        fmt.Println(v)
    }(v)
}
```

```go
// NG: 無制限 goroutine スポーン
for _, item := range millionItems {
    go process(item) // OOM リスク
}

// OK: semaphore でスロットリング
sem := make(chan struct{}, 100)
for _, item := range millionItems {
    sem <- struct{}{}
    go func(item Item) {
        defer func() { <-sem }()
        process(item)
    }(item)
}
```

### MEDIUM

```go
// NG: エラー集約が必要な goroutine に WaitGroup
// OK: errgroup を使う
g, ctx := errgroup.WithContext(ctx)
for _, item := range items {
    item := item
    g.Go(func() error {
        return process(ctx, item)
    })
}
if err := g.Wait(); err != nil { ... }
```

---

## カテゴリ4: Channel パターン

### CRITICAL

```go
// NG: nil channel への送信（永久ブロック）
var ch chan int
ch <- 42 // デッドロック

// NG: 同一チャネルの二重クローズ（panic）
close(ch)
close(ch) // panic: close of closed channel
```

### HIGH

```go
// NG: レシーバー側からチャネルをクローズ（センダーが panic）
func consumer(ch chan<- int) {
    close(ch) // センダーが次に送信すると panic
}

// OK: センダー側がクローズ、またはシグナル用の別チャネル
```

```go
// NG: Ticker を defer で Stop しない
ticker := time.NewTicker(time.Second)
go func() {
    for range ticker.C { ... }
}()
// ticker.Stop() なし → goroutine と memory leak

// OK
ticker := time.NewTicker(time.Second)
defer ticker.Stop()
```

```go
// NG: 閉じたチャネルから読む時に ok を確認しない
for v := range ch { ... }   // 実は ok チェックが暗黙に入っている（range は OK）
v := <-ch                    // NG: closed channel からゼロ値を受け取っても気づかない

// OK
v, ok := <-ch
if !ok { return } // チャネルが閉じた
```

---

## カテゴリ5: Context

### CRITICAL

```go
// NG: cancel を defer しない（context leak）
ctx, cancel := context.WithCancel(parent)
// cancel() が呼ばれない → goroutine とリソースがリーク

// OK
ctx, cancel := context.WithCancel(parent)
defer cancel()
```

### HIGH

```go
// NG: r.Context() を使わず独自コンテキスト（キャンセル不伝播）
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := context.Background() // リクエストのキャンセルが伝播しない
    result, err := db.QueryContext(ctx, query)
    ...
}

// OK
func handler(w http.ResponseWriter, r *http.Request) {
    result, err := db.QueryContext(r.Context(), query)
    ...
}
```

```go
// NG: 長時間ループで ctx.Done() をチェックしない
for _, item := range items {
    process(item) // ctx がキャンセルされても止まらない
}

// OK
for _, item := range items {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
    }
    process(item)
}
```

```go
// NG: context を struct フィールドに格納
type Service struct {
    ctx context.Context // NG: Go 仕様では関数の第1引数として渡す
}

// OK: 関数シグネチャで渡す
func (s *Service) Process(ctx context.Context, data []byte) error { ... }
```

```go
// NG: context を受け取る関数内で context.Background() を生成（伝播チェーンが切れる）
func doWork(ctx context.Context) {
    child := context.Background() // NG: 親のキャンセルが伝播しない
    db.QueryContext(child, query)
}

// OK: 受け取った ctx を渡す
func doWork(ctx context.Context) {
    db.QueryContext(ctx, query)
}
```

### MEDIUM

```go
// NG: 必須パラメータを context に詰め込む
ctx = context.WithValue(ctx, "userID", userID) // 型安全でない、関数シグネチャに明示すべき

// OK: 認証情報など横断的関心事のみ context に
```

---

## カテゴリ6: インターフェース設計

### HIGH

```go
// NG: 実装パッケージ側でインターフェースを定義（Go は消費側で定義が慣習）
// storage/storage.go
type Storage interface { // NG: 実装パッケージに定義
    Get(key string) ([]byte, error)
    Set(key string, value []byte) error
}

// OK: 消費側（使う側）でインターフェースを定義
// handler/handler.go
type storage interface { // 非公開で十分
    Get(key string) ([]byte, error)
    Set(key string, value []byte) error
}
```

```go
// NG: 大きすぎるインターフェース
type Repository interface {
    GetUser(id int) (*User, error)
    CreateUser(u *User) error
    UpdateUser(u *User) error
    DeleteUser(id int) error
    GetPost(id int) (*Post, error)
    CreatePost(p *Post) error
    // ... 10メソッド以上
}

// OK: 小さく分割
type UserReader interface { GetUser(id int) (*User, error) }
type UserWriter interface { CreateUser(u *User) error; UpdateUser(u *User) error }
```

```go
// NG: コンパイル時チェックなし
// OK: インターフェース実装の確認
var _ io.Reader = (*MyReader)(nil)
```

---

## カテゴリ7: Generics（Go 1.18+）

### HIGH

```go
// NG: 制約なし型パラメータで数値演算
func Sum[T any](items []T) T { // any だと + 演算子が使えない
    var total T
    for _, v := range items { total += v } // コンパイルエラー
    return total
}

// OK: constraints を使う
func Sum[T constraints.Integer | constraints.Float](items []T) T {
    var total T
    for _, v := range items { total += v }
    return total
}
```

```go
// NG: slices/maps stdlib（1.21+）を再実装
func Contains[T comparable](slice []T, elem T) bool {
    for _, v := range slice {
        if v == elem { return true }
    }
    return false
}

// OK: 標準ライブラリを使う（Go 1.21+）
import "slices"
slices.Contains(slice, elem)
```

---

## カテゴリ8: メモリ・パフォーマンス

### CRITICAL

```go
// NG: sync.Mutex を値レシーバーで使用（コピーでロックが機能しない）
func (c Counter) Increment() { // 値レシーバー
    c.mu.Lock()   // c がコピーされているためロックが効かない
    c.count++
    c.mu.Unlock()
}

// OK: ポインタレシーバー
func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}
```

### HIGH

```go
// NG: サイズ既知の map をプリサイズしない
m := make(map[string]int) // 再ハッシュが多発
for _, item := range millionItems {
    m[item.Key] = item.Value
}

// OK: 事前確保
m := make(map[string]int, len(millionItems))
```

```go
// NG: defer をループ内で使用（関数が返るまで全て蓄積）
for _, path := range paths {
    f, _ := os.Open(path)
    defer f.Close() // ループが終わるまで全ファイルが開きっぱなし
}

// OK: 即座に閉じるか、別関数に切り出す
for _, path := range paths {
    if err := processFile(path); err != nil { ... }
}
func processFile(path string) error {
    f, err := os.Open(path)
    if err != nil { return err }
    defer f.Close() // 関数終了時にクローズ
    ...
}
```

```go
// NG: スライスの容量を事前確保しない
var result []string
for _, item := range items {
    result = append(result, transform(item)) // 再アロケーションが多発
}

// OK
result := make([]string, 0, len(items))
for _, item := range items {
    result = append(result, transform(item))
}
```

---

## カテゴリ9: JSON・エンコーディング

### HIGH

```go
// NG: 非公開フィールドが json.Marshal で無言でスキップ（デバッグ困難）
type User struct {
    id   int    // 小文字 → JSON に含まれない、エラーも出ない
    Name string
}

// OK: 公開フィールドを使うか、MarshalJSON を実装
```

```go
// NG: int64 が JSON で float64 として精度を失う
// JavaScript の Number は 2^53 まで精度を保証。それを超える int64 は精度喪失
type Order struct {
    ID int64 `json:"id"` // 大きな値で JavaScript 側が不正確
}

// OK: 文字列にエンコード
type Order struct {
    ID int64 `json:"id,string"` // "id": "9007199254740993"
}
```

```go
// NG: 任意数値を interface{} でデコード → float64 に変換されて精度喪失
var v interface{}
json.Unmarshal(data, &v)
m := v.(map[string]interface{})
id := m["id"].(float64) // 精度喪失

// OK: json.Number を使う
decoder := json.NewDecoder(r)
decoder.UseNumber()
```

### MEDIUM

```go
// NG: omitempty と pointer の組み合わせミス
type Config struct {
    Enabled *bool `json:"enabled,omitempty"` // false は省略されない（pointer の場合）
    // *bool の場合: nil → 省略, &false → {"enabled": false}
    // bool の場合: false → 省略（これが罠）
}
// bool の omitempty は false を省略してしまう → *bool を使う
```

---

## カテゴリ10: HTTP・ネットワーク

### CRITICAL

```go
// NG: http.Client にタイムアウト未設定（Slowloris 攻撃で goroutine leak）
client := &http.Client{} // タイムアウトなし

// OK
client := &http.Client{
    Timeout: 30 * time.Second,
}
```

### HIGH

```go
// NG: http.DefaultServeMux に /debug/pprof が公開される
http.ListenAndServe(":8080", nil) // nil は DefaultServeMux を使う
// → import _ "net/http/pprof" があると /debug/pprof がパブリックに公開

// OK: 専用の Mux を使う
mux := http.NewServeMux()
http.ListenAndServe(":8080", mux)
```

```go
// NG: http.Client をリクエストごとに新規作成（TCP 接続プールが効かない）
func fetchData(url string) (*Data, error) {
    client := &http.Client{} // 毎回新規作成
    resp, err := client.Get(url)
    ...
}

// OK: パッケージレベルかアプリレベルで共有
var httpClient = &http.Client{Timeout: 30 * time.Second}
```

```go
// NG: TLS 証明書検証を無効化
tlsConfig := &tls.Config{InsecureSkipVerify: true} // 中間者攻撃の危険
```

```go
// NG: レスポンスボディを読み切らずにクローズ（接続再利用不可）
resp, _ := client.Get(url)
defer resp.Body.Close()
// Body を読まずに Close → HTTP Keep-Alive が機能しない

// OK: 読み切ってから Close
defer func() {
    io.Copy(io.Discard, resp.Body)
    resp.Body.Close()
}()
```

---

## カテゴリ11: セキュリティ

### CRITICAL

```go
// NG: math/rand でセキュリティトークン生成
token := fmt.Sprintf("%d", rand.Int63()) // 予測可能

// OK: crypto/rand
b := make([]byte, 32)
if _, err := rand.Read(b); err != nil { ... }
token := hex.EncodeToString(b)
```

```go
// NG: SQL インジェクション
query := fmt.Sprintf("SELECT * FROM users WHERE id=%d AND name='%s'", id, name)
db.Query(query)

// OK: プレースホルダを使う
db.Query("SELECT * FROM users WHERE id=? AND name=?", id, name)
```

```go
// NG: filepath.Join を使わないパス構築（path traversal）
userFile := "/uploads/" + username + "/" + filename
// username = "../../../etc/passwd" → path traversal

// OK
userFile := filepath.Join("/uploads", username, filename)
// さらに filepath.Rel で /uploads 外へのエスケープを確認
```

---

## カテゴリ12: テスト

### HIGH

```go
// NG: Table-driven test を使わない
func TestAdd(t *testing.T) {
    if Add(1, 2) != 3 { t.Error("failed") }
    if Add(0, 0) != 0 { t.Error("failed") }
    if Add(-1, 1) != 0 { t.Error("failed") }
}

// OK
func TestAdd(t *testing.T) {
    tests := []struct {
        a, b, want int
    }{
        {1, 2, 3},
        {0, 0, 0},
        {-1, 1, 0},
    }
    for _, tt := range tests {
        t.Run(fmt.Sprintf("%d+%d", tt.a, tt.b), func(t *testing.T) {
            if got := Add(tt.a, tt.b); got != tt.want {
                t.Errorf("Add(%d, %d) = %d, want %d", tt.a, tt.b, got, tt.want)
            }
        })
    }
}
```

```go
// NG: t.Fatal を goroutine 内から呼ぶ（panic を引き起こす）
go func() {
    if err != nil {
        t.Fatal(err) // NG: goroutine 内からは呼べない
    }
}()

// OK: チャネルでエラーを返す
errCh := make(chan error, 1)
go func() {
    errCh <- doWork()
}()
if err := <-errCh; err != nil {
    t.Fatal(err)
}
```

```go
// NG: テスト内の非同期処理を time.Sleep で待つ（flaky test）
go startServer()
time.Sleep(100 * time.Millisecond) // CI 環境で落ちる

// OK: チャネルや condition で準備完了を通知
ready := make(chan struct{})
go startServer(ready)
<-ready
```

### MEDIUM

```go
// NG: テストヘルパーで t.Helper() を呼ばない（失敗時の行番号が不正確）
func assertEqual(t *testing.T, got, want int) {
    // t.Helper() なし → テストヘルパーの行番号が報告される
    if got != want { t.Errorf(...) }
}

// OK
func assertEqual(t *testing.T, got, want int) {
    t.Helper()
    if got != want { t.Errorf("got %d, want %d", got, want) }
}
```

---

## カテゴリ13: モジュール・エコシステム

### HIGH

```go
// NG: ローカルパス replace がリリースに残っている
// go.mod
replace github.com/example/lib => ../local-lib // リリース前に削除必須
```

```go
// NG: go.sum を git コミットしない
// .gitignore に go.sum が含まれている → 再現性・セキュリティ問題
// go.sum は必ずコミットすること
```

```go
// NG: go.mod の go バージョンが古すぎる
module myapp

go 1.13 // セキュリティ fix や新機能が使えない
// OK: 少なくとも go 1.21 以上（サポート対象の最小バージョン）
```

### MEDIUM

```go
// NG: internal/ パッケージの設計違反（モジュール外から使う意図）
// モジュール external には mymodule/internal/... をインポートできない
// internal/ は「このモジュールのプライベート実装」を示す設計上の意図
```
