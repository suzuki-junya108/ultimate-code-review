# PHP 言語固有ディープレビュー

**品質基準**: PHP コアコントリビューター・OWASP PHP Security チートシートレベル。
PHPStan level 9 をパスしても残る問題を検出する。

---

## カテゴリ1: セキュリティ Critical（PHP 固有 RCE/注入）

### CRITICAL

```php
// NG: eval でユーザー入力を実行
eval($userInput); // 任意コード実行

// OK: eval は基本的に使わない。動的処理が必要なら設計を見直す
```

```php
// NG: exec/shell_exec にユーザー入力を渡す
exec("ls -la " . $_GET['path']); // コマンドインジェクション

// OK: コマンド実行を避けるか、引数をエスケープ
$safePath = escapeshellarg($_GET['path']);
exec("ls -la " . $safePath);
// ただし escapeshellarg でもすべてのケースが安全とは限らないため
// ユーザー入力をシェルコマンドに使わない設計が最善
```

```php
// NG: extract($_POST) で任意変数上書き
extract($_POST); // $admin = true 等を注入できる

// OK: 個別にアクセス
$name = $_POST['name'] ?? '';
$email = $_POST['email'] ?? '';
```

```php
// NG: include にユーザー入力パス（LFI/RFI）
include($_GET['page'] . '.php'); // ../../../etc/passwd 等でファイル読み取り

// OK: ホワイトリストで制限
$allowedPages = ['home', 'about', 'contact'];
$page = $_GET['page'] ?? 'home';
if (!in_array($page, $allowedPages, true)) {
    $page = 'home';
}
include('pages/' . $page . '.php');
```

```php
// NG: unserialize でユーザーデータをデシリアライズ（PHP Object Injection → RCE）
$obj = unserialize($_COOKIE['data']); // PHPGGC gadget chain で RCE 可能

// OK: JSON を使う
$data = json_decode($_COOKIE['data'], true);
// または allowed_classes でクラスを制限
$obj = unserialize($data, ['allowed_classes' => false]);
```

---

## カテゴリ2: セキュリティ High（認証・セッション）

### HIGH

```php
// NG: md5/sha1 でパスワードハッシュ
$hash = md5($password); // 虹の表攻撃に脆弱

// OK: password_hash を使う
$hash = password_hash($password, PASSWORD_ARGON2ID);
// 検証
if (password_verify($password, $hash)) { ... }
```

```php
// NG: mt_rand でセキュリティトークン生成
$token = md5(mt_rand()); // 予測可能

// OK: random_bytes を使う
$token = bin2hex(random_bytes(32)); // 暗号学的に安全な乱数
```

```php
// NG: ログイン後にセッション固定化攻撃への対策なし
// ログイン前に取得したセッション ID を攻撃者が知っていると、ログイン後も使い回せる

// OK: ログイン成功後にセッション ID を再生成
session_start();
// ... 認証処理 ...
session_regenerate_id(true); // true で古いセッションファイルを削除
```

```php
// NG: strcmp でパスワード/トークンを比較（タイミング攻撃）
if (strcmp($userToken, $expectedToken) === 0) { ... }
// 文字列の一致位置まで時間が短く、タイミングで推測される可能性

// OK: hash_equals でタイミング安全な比較
if (hash_equals($expectedToken, $userToken)) { ... }
```

```php
// NG: ファイルアップロードで拡張子のみ検証
$ext = pathinfo($_FILES['file']['name'], PATHINFO_EXTENSION);
if (in_array($ext, ['jpg', 'png', 'gif'])) { ... }
// .php.jpg のようなファイルや Content-Type 偽装が通る

// OK: 実際の MIME type を finfo で確認
$finfo = new finfo(FILEINFO_MIME_TYPE);
$mimeType = $finfo->file($_FILES['file']['tmp_name']);
$allowedMimes = ['image/jpeg', 'image/png', 'image/gif'];
if (!in_array($mimeType, $allowedMimes, true)) {
    throw new InvalidFileException('Invalid file type');
}
```

```php
// NG: header Location にユーザー入力を直接使用（Open Redirect）
header('Location: ' . $_GET['redirect']); // 任意サイトへのリダイレクト

// OK: 許可リストで制限
$allowedRedirects = ['/home', '/profile', '/dashboard'];
$redirect = $_GET['redirect'] ?? '/home';
if (!in_array($redirect, $allowedRedirects, true)) {
    $redirect = '/home';
}
header('Location: ' . $redirect);
```

---

## カテゴリ3: 型安全（PHP 8.x）

### CRITICAL

```php
// NG: == 比較の型ジャグリング（予期しない true）
var_dump("0" == false);   // true
var_dump("" == false);    // true
var_dump(0 == "foo");     // true（PHP 7 以前）
var_dump(null == false);  // true
var_dump("1" == "01");    // true（数値として比較）

// OK: === で厳密比較
if ($status === false) { ... }
if ($count === 0) { ... }
```

### HIGH

```php
// NG: 関数シグネチャに型宣言なし
function processUser($id, $data) { // $id が int か string か不明
    return updateRecord($id, $data);
}

// OK: 型宣言
function processUser(int $id, array $data): bool {
    return updateRecord($id, $data);
}
```

```php
// NG: declare(strict_types=1) なし（暗黙の型強制）
// strict_types なしだと文字列 "42" が int パラメータに渡せてしまう
processUser("42", $data); // 暗黙に int(42) に変換される

// OK: ファイル先頭に追加
<?php
declare(strict_types=1);
```

```php
// NG: mixed 型を公開 API に使用
public function process(mixed $input): mixed { ... }
// 呼び出し側が型を把握できない

// OK: 具体的な型を使う
public function process(array|string $input): ProcessResult { ... }
```

---

## カテゴリ4: エラーハンドリング

### HIGH

```php
// NG: @ エラー抑制演算子の使用
$result = @file_get_contents($url); // エラーが完全に隠れる

// OK: エラーハンドリングを明示
$result = file_get_contents($url);
if ($result === false) {
    throw new NetworkException("Failed to fetch: $url");
}
```

```php
// NG: 空の catch ブロック（例外を飲み込む）
try {
    $db->execute($query);
} catch (PDOException $e) {
    // 何もしない
}

// OK: 少なくともログを出力
try {
    $db->execute($query);
} catch (PDOException $e) {
    $this->logger->error('DB query failed', ['exception' => $e]);
    throw new DatabaseException('Query failed', 0, $e);
}
```

```php
// NG: die()/exit() をライブラリコードで使用
function validateToken(string $token): void {
    if (!isValid($token)) {
        die('Invalid token'); // 呼び出し元がコントロールできない
    }
}

// OK: 例外を投げる
function validateToken(string $token): void {
    if (!isValid($token)) {
        throw new InvalidTokenException('Invalid token');
    }
}
```

---

## カテゴリ5: データベース

### CRITICAL

```php
// NG: PDO PreparedStatement を使わずクエリ文字列を結合（SQL インジェクション）
$query = "SELECT * FROM users WHERE id=" . $_GET['id'];
$stmt = $pdo->query($query);

// OK: prepared statement
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = :id");
$stmt->execute([':id' => (int)$_GET['id']]);
$user = $stmt->fetch();
```

### HIGH

```php
// NG: N+1 クエリ（ループ内でクエリ発行）
$users = $pdo->query("SELECT * FROM users")->fetchAll();
foreach ($users as $user) {
    $posts = $pdo->prepare("SELECT * FROM posts WHERE user_id = ?")->execute([$user['id']]); // N+1
}

// OK: JOIN か IN を使う
$stmt = $pdo->prepare("
    SELECT u.*, p.title
    FROM users u
    LEFT JOIN posts p ON p.user_id = u.id
");
$stmt->execute();
```

```php
// NG: LIMIT なしの全件取得（メモリ枯渇リスク）
$stmt = $pdo->query("SELECT * FROM events"); // テーブルが100万行になったら?
$events = $stmt->fetchAll();

// OK: LIMIT + OFFSET か cursor ベースのページネーション
$stmt = $pdo->prepare("SELECT * FROM events ORDER BY id LIMIT :limit OFFSET :offset");
$stmt->bindValue(':limit', $pageSize, PDO::PARAM_INT);
$stmt->bindValue(':offset', $page * $pageSize, PDO::PARAM_INT);
```

```php
// NG: トランザクション内で例外ハンドリングなし
$pdo->beginTransaction();
$pdo->exec("UPDATE accounts SET balance = balance - 100 WHERE id = 1");
$pdo->exec("UPDATE accounts SET balance = balance + 100 WHERE id = 2");
$pdo->commit(); // 2行目で例外が発生したら中途状態でコミットされる可能性

// OK: try-catch でロールバック
$pdo->beginTransaction();
try {
    $pdo->exec("UPDATE accounts SET balance = balance - 100 WHERE id = 1");
    $pdo->exec("UPDATE accounts SET balance = balance + 100 WHERE id = 2");
    $pdo->commit();
} catch (Exception $e) {
    $pdo->rollBack();
    throw $e;
}
```

---

## カテゴリ6: モダン PHP イディオム（PHP 8.x）

### HIGH

```php
// NG: match 式が使えるのに switch + break を使用
switch ($status) {
    case 'active':
        $label = '有効';
        break;
    case 'inactive':
        $label = '無効';
        break;
    default:
        $label = '不明';
}
// fall-through バグのリスクがある

// OK: match（厳密比較、fall-through なし）
$label = match($status) {
    'active' => '有効',
    'inactive' => '無効',
    default => '不明',
};
```

```php
// NG: PHP 8.1 の enum を使えるのに定数で状態を表現
class Status {
    const ACTIVE = 'active';
    const INACTIVE = 'inactive';
    const PENDING = 'pending';
}
// 任意の string が Status として渡せてしまう

// OK: enum（PHP 8.1+）
enum Status: string {
    case Active = 'active';
    case Inactive = 'inactive';
    case Pending = 'pending';
}
function process(Status $status): void { ... }
```

```php
// NG: 多段 null チェック
if ($user !== null && $user->getProfile() !== null && $user->getProfile()->getAddress() !== null) {
    $city = $user->getProfile()->getAddress()->getCity();
}

// OK: Nullsafe operator（PHP 8+）
$city = $user?->getProfile()?->getAddress()?->getCity();
```

### MEDIUM

```php
// NG: bool フラグだらけの関数呼び出し
createUser($name, $email, true, false, true); // 引数の意味が不明

// OK: Named arguments（PHP 8+）
createUser(
    name: $name,
    email: $email,
    isAdmin: true,
    sendWelcomeEmail: false,
    requireEmailVerification: true,
);
```

---

## カテゴリ7: PSR・エコシステム準拠

### HIGH

```php
// NG: Composer autoload を使わず手動 require
require_once 'classes/User.php';
require_once 'classes/Database.php';
require_once 'helpers/utils.php';

// OK: Composer PSR-4 autoload を設定
// composer.json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
// コード内
use App\Models\User;
use App\Services\Database;
```

```php
// NG: 依存関係を直接インスタンス化（DI なし）
class UserService {
    private Database $db;

    public function __construct() {
        $this->db = new Database(); // テスト不可、結合が強い
    }
}

// OK: DI コンテナ経由でインジェクション
class UserService {
    public function __construct(
        private readonly DatabaseInterface $db
    ) {}
}
```

```json
// NG: composer.json に php バージョン制約なし
{
    "name": "example/app"
    // "require" に php バージョンなし
}

// OK
{
    "require": {
        "php": ">=8.1"
    }
}
```

---

## カテゴリ8: フレームワーク固有（条件付き）

### Laravel 検出時

```php
// NG: env() を config ファイル外で直接呼ぶ（設定キャッシュ時に動作しない）
$dbHost = env('DB_HOST', 'localhost'); // php artisan config:cache 後に null になる

// OK: config() を使う
$dbHost = config('database.connections.mysql.host');
```

```php
// NG: N+1 クエリ（with() eager loading 未使用）
$users = User::all();
foreach ($users as $user) {
    echo $user->profile->name; // N+1: ユーザーごとに profile を SELECT
}

// OK: eager loading
$users = User::with('profile')->get();
foreach ($users as $user) {
    echo $user->profile->name;
}
```

```php
// NG: dd() / dump() が本番コードに残っている
public function process(Request $request): Response {
    dd($request->all()); // デバッグコードが残存
    return response()->json($result);
}
```

### Symfony 検出時

```php
// NG: Service definition の public: true 乱用
# config/services.yaml
services:
    App\Service\UserService:
        public: true  # コンテナから直接取得できてしまう（テスト以外では不要）

# OK: デフォルトの private のまま使う（autowiring で自動注入）
```

### WordPress 検出時

```php
// NG: $wpdb->prepare() なしで SQL クエリを実行
global $wpdb;
$results = $wpdb->get_results("SELECT * FROM {$wpdb->posts} WHERE post_author = $user_id");

// OK: prepare() でプレースホルダを使う
$results = $wpdb->get_results(
    $wpdb->prepare("SELECT * FROM {$wpdb->posts} WHERE post_author = %d", $user_id)
);
```

```php
// NG: nonce 検証なしのフォーム処理
function handle_form_submission() {
    if (isset($_POST['data'])) {
        // CSRF 保護なし
        process_data($_POST['data']);
    }
}

// OK: nonce を検証
function handle_form_submission() {
    if (!isset($_POST['_wpnonce']) || !wp_verify_nonce($_POST['_wpnonce'], 'my_action')) {
        wp_die('Security check failed');
    }
    process_data($_POST['data']);
}
```
