# Java 言語固有ディープレビュー

**品質基準**: Oracle Java チャンピオン・JVM エンジニアレベル。
SpotBugs / PMD / Checkstyle をパスしても残る問題を検出する。

---

## カテゴリ1: Java 21 モダンイディオム

### CRITICAL

```java
// NG: java.util.Date / Calendar を新規コードで使用
Date now = new Date(); // 非推奨、タイムゾーン問題が山積み
Calendar cal = Calendar.getInstance();
cal.add(Calendar.DAY_OF_MONTH, 7);

// OK: java.time.*
LocalDate today = LocalDate.now();
ZonedDateTime deadline = ZonedDateTime.now(ZoneId.of("Asia/Tokyo")).plusDays(7);
```

```java
// NG: String の == 比較（インターンに依存した偶然の動作）
if (status == "active") { ... } // new String("active") と比較すると false

// OK
if ("active".equals(status)) { ... } // null safe な equals
```

### HIGH

```java
// NG: raw types の使用
List list = new ArrayList(); // 型安全でない
list.add("string");
list.add(42); // コンパイラが警告しない（エラーにもならない）

// OK: generics を使う
List<String> list = new ArrayList<>();
```

```java
// NG: Optional.get() を isPresent() チェックなしで呼ぶ
Optional<User> user = findUser(id);
String name = user.get().getName(); // NoSuchElementException のリスク

// OK: orElse / orElseThrow を使う
String name = findUser(id)
    .map(User::getName)
    .orElseThrow(() -> new UserNotFoundException(id));
```

```java
// NG: instanceof 後に明示的キャスト（Java 16+ はパターン変数を使う）
if (obj instanceof String) {
    String s = (String) obj; // 二重チェック
    process(s);
}

// OK: パターン変数（Java 16+）
if (obj instanceof String s) {
    process(s);
}
```

```java
// NG: switch 文の fall-through バグ
switch (status) {
    case "active":
        doActive();
        // break 忘れ → "inactive" の処理も実行される
    case "inactive":
        doInactive();
        break;
}

// OK: switch 式（Java 14+）
String result = switch (status) {
    case "active" -> doActive();
    case "inactive" -> doInactive();
    default -> throw new IllegalStateException("Unknown: " + status);
};
```

---

## カテゴリ2: 最新言語機能（Java 16-21）

### HIGH

```java
// NG: getter-only クラスを手動定義（Java 16+ は record を使う）
public class Point {
    private final int x;
    private final int y;
    public Point(int x, int y) { this.x = x; this.y = y; }
    public int x() { return x; }
    public int y() { return y; }
    @Override public boolean equals(Object o) { ... } // 大量のボイラープレート
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
}

// OK: Record（Java 16+）
public record Point(int x, int y) {} // equals/hashCode/toString 自動生成
```

```java
// NG: if-instanceof チェーン（Java 21 は Pattern Matching for switch を使う）
Object shape = getShape();
if (shape instanceof Circle c) {
    return Math.PI * c.radius() * c.radius();
} else if (shape instanceof Rectangle r) {
    return r.width() * r.height();
} else {
    throw new IllegalArgumentException("Unknown shape");
}

// OK: Pattern matching for switch（Java 21）
return switch (shape) {
    case Circle c -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    default -> throw new IllegalArgumentException("Unknown shape");
};
```

```java
// NG: Platform thread でブロッキング I/O（Java 21 は Virtual threads を使う）
ExecutorService executor = Executors.newFixedThreadPool(200);
executor.submit(() -> {
    String result = blockingHttpCall(); // 200スレッドが全部 I/O 待ちで詰まる
});

// OK: Virtual threads（Java 21）
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
executor.submit(() -> {
    String result = blockingHttpCall(); // Virtual thread なので JVM がブロックを管理
});
```

---

## カテゴリ3: Stream・Lambda（Java 8+）

### HIGH

```java
// NG: Stream 操作内の副作用（parallel stream で race condition）
List<String> results = new ArrayList<>();
items.parallelStream().forEach(item -> {
    results.add(process(item)); // ArrayList は thread-safe でない
});

// OK: collect を使う
List<String> results = items.parallelStream()
    .map(this::process)
    .collect(Collectors.toList());
```

```java
// NG: Java 16+ で Collectors.toList() を使用
List<String> names = users.stream()
    .map(User::getName)
    .collect(Collectors.toList()); // mutable list を返す

// OK: .toList()（Java 16+）を使う（immutable）
List<String> names = users.stream()
    .map(User::getName)
    .toList();
```

```java
// NG: CPU バウンドでない処理や I/O に parallelStream()
List<User> users = ids.parallelStream()
    .map(id -> userRepository.findById(id)) // DB 呼び出しに parallel は逆効果
    .collect(toList());

// OK: 通常の stream か、CompletableFuture で非同期化
```

---

## カテゴリ4: 並行処理

### CRITICAL

```java
// NG: double-checked locking で volatile フィールドを使わない（Java メモリモデル違反）
private static Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {
        synchronized (Singleton.class) {
            if (instance == null) {
                instance = new Singleton(); // volatile なし → 部分的に初期化されたオブジェクトが見える可能性
            }
        }
    }
    return instance;
}

// OK: volatile を付ける
private static volatile Singleton instance;
// または: Initialization-on-Demand Holder パターン
private static class Holder {
    static final Singleton INSTANCE = new Singleton();
}
public static Singleton getInstance() { return Holder.INSTANCE; }
```

```java
// NG: HashMap を複数スレッドで共有（無限ループの可能性）
private static Map<String, Integer> cache = new HashMap<>(); // NG: thread-safe でない
// 並行変更により resize 中に無限ループが発生することがある（Java 7 以前で顕著）

// OK
private static Map<String, Integer> cache = new ConcurrentHashMap<>();
```

### HIGH

```java
// NG: ExecutorService を shutdown なしで使う（thread leak）
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(task);
// shutdown() なし → スレッドが残り続ける

// OK
ExecutorService executor = Executors.newFixedThreadPool(10);
try {
    executor.submit(task);
} finally {
    executor.shutdown();
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        executor.shutdownNow();
    }
}
```

```java
// NG: Future.get() をタイムアウトなしで呼ぶ
String result = future.get(); // 永久ブロックの可能性

// OK: タイムアウトを指定
String result = future.get(30, TimeUnit.SECONDS);
```

```java
// NG: wait() のスプリアスウェイクアップ対策なし
synchronized (lock) {
    lock.wait(); // スプリアスウェイクアップで条件が満たされていなくても起きる可能性
    doWork();
}

// OK: while ループで条件を確認
synchronized (lock) {
    while (!condition) {
        lock.wait();
    }
    doWork();
}
```

---

## カテゴリ5: メモリ・GC

### CRITICAL

```java
// NG: finalize() メソッドのオーバーライド（Java 9 deprecated、Java 18 で削除）
@Override
protected void finalize() throws Throwable {
    resource.close(); // GC タイミングが不定、デッドロックの可能性
    super.finalize();
}

// OK: java.lang.ref.Cleaner（Java 9+）か try-with-resources
private final Cleaner.Cleanable cleanable;
public MyResource() {
    cleanable = cleaner.register(this, new CleanupAction(resource));
}
```

### HIGH

```java
// NG: 非 static な内部クラスが外部クラスの参照を暗黙に保持
public class Outer {
    private byte[] largeData = new byte[10 * 1024 * 1024]; // 10MB

    public class Inner { // 非 static → Outer への暗黙の参照を保持
        public void doWork() { ... }
    }
}
// Inner を保持し続けると Outer（10MB）も GC されない

// OK: static inner class
public static class Inner {
    public void doWork() { ... }
}
```

```java
// NG: ThreadLocal を remove() なしで使用（thread pool でメモリリーク）
private static final ThreadLocal<Context> context = new ThreadLocal<>();

public void process() {
    context.set(new Context());
    doWork();
    // remove() なし → thread pool のスレッドが再利用されると前回の Context が残る
}

// OK
public void process() {
    context.set(new Context());
    try {
        doWork();
    } finally {
        context.remove();
    }
}
```

---

## カテゴリ6: 例外・エラーハンドリング

### CRITICAL

```java
// NG: Throwable や Exception を catch（OOM 等の Error まで捕まえる）
try {
    doWork();
} catch (Throwable t) { // OutOfMemoryError も捕まえてしまう
    log.error("error", t);
}

// OK: 具体的な例外を catch
try {
    doWork();
} catch (IOException | ParseException e) {
    log.error("処理に失敗しました", e);
    throw new ServiceException("処理失敗", e);
}
```

### HIGH

```java
// NG: Checked exception を空の catch で握りつぶす
try {
    connection.close();
} catch (SQLException e) {
    // 何もしない → 問題が隠蔽される
}

// OK: 少なくともログを出力
try {
    connection.close();
} catch (SQLException e) {
    log.warn("Connection close failed", e);
}
```

```java
// NG: 例外を log して再 throw（スタックトレースが2重に出る）
try {
    doWork();
} catch (Exception e) {
    log.error("error", e); // ログに記録
    throw e; // 再 throw すると上位でもログが出て重複する
}

// OK: ログするか再 throw するかどちらか一方
// 再 throw する場合はラップして context を追加
throw new ServiceException("doWork failed", e);
```

---

## カテゴリ7: セキュリティ（Java 固有）

### CRITICAL

```java
// NG: ObjectInputStream.readObject() を未信頼データに使用（デシリアライゼーション RCE）
ObjectInputStream ois = new ObjectInputStream(userProvidedStream);
Object obj = ois.readObject(); // PHPGGC / ysoserial 等で RCE 可能

// OK: JSON 等の安全なフォーマットを使う、またはホワイトリスト検証
// resolveClass をオーバーライドして許可クラスのみデシリアライズ
```

### HIGH

```java
// NG: ProcessBuilder / Runtime.exec() にユーザー入力を渡す
String cmd = "ls -la " + userInput; // コマンドインジェクション
Runtime.getRuntime().exec(cmd);

// OK: 引数を配列で渡し、シェル経由を避ける
ProcessBuilder pb = new ProcessBuilder("ls", "-la", sanitizedPath);
pb.start();
```

```java
// NG: JDBC で文字列結合（SQL インジェクション）
String sql = "SELECT * FROM users WHERE name='" + userName + "'";
statement.execute(sql);

// OK: PreparedStatement
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM users WHERE name=?"
);
ps.setString(1, userName);
ps.executeQuery();
```

```java
// NG: java.util.Random をセキュリティ用途に使用
String token = Long.toHexString(new Random().nextLong()); // 予測可能

// OK: SecureRandom
byte[] bytes = new byte[32];
new SecureRandom().nextBytes(bytes);
String token = Base64.getEncoder().encodeToString(bytes);
```

### MEDIUM

```java
// NG: ログに個人情報を無意識に出力
log.info("User login: {}", user.toString()); // user.toString() にメールや電話番号が含まれる可能性

// OK: ログ出力用の toString() を設計、または個人情報フィールドを明示的に除外
log.info("User login: userId={}", user.getId());
```
