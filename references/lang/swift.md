# Swift 言語固有ディープレビュー

**品質基準**: Apple SDK エンジニア・WWDC セッション登壇者レベル。
SwiftLint をパスしても残る問題を検出する。

---

## カテゴリ1: ARC メモリ管理

### CRITICAL

```swift
// NG: クロージャが self を強参照し、self もクロージャを強参照（循環参照でメモリリーク）
class ViewController: UIViewController {
    var completion: (() -> Void)?

    func setup() {
        completion = {
            self.updateUI() // strong capture → 循環参照
        }
    }
}

// OK: [weak self] でキャプチャ
func setup() {
    completion = { [weak self] in
        self?.updateUI()
    }
}
```

```swift
// NG: [unowned self] を self が解放される可能性のあるクロージャで使用
class DataLoader {
    var onComplete: (() -> Void)?

    func load() {
        networkTask { [unowned self] in // self が解放されていたらクラッシュ
            self.updateUI()
        }
    }
}

// OK: [weak self] + nil チェック
func load() {
    networkTask { [weak self] in
        guard let self else { return }
        self.updateUI()
    }
}
```

### HIGH

```swift
// NG: delegate パターンで weak var にしない（循環参照）
class ListView: UIView {
    var delegate: ListViewDelegate? // strong reference → 循環参照の可能性

    // OK
    weak var delegate: ListViewDelegate?
}
```

```swift
// NG: クロージャの capture list を省略（self が暗黙に強参照でキャプチャ）
class MyClass {
    func startTimer() {
        Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { timer in
            self.update() // capture list なし → strong capture
        }
    }
}

// OK
func startTimer() {
    Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] timer in
        self?.update()
    }
}
```

---

## カテゴリ2: Optional の安全な扱い

### CRITICAL

```swift
// NG: ! 強制アンラップをランダムな場所で使用
let user = findUser(id: userId)!
let name = user.profile!.name! // nil 時に即クラッシュ

// OK: guard let か if let でアンラップ
guard let user = findUser(id: userId),
      let name = user.profile?.name else {
    return
}
```

```swift
// NG: try! を production コードで使用
let data = try! jsonEncoder.encode(model) // エラー時に即クラッシュ

// OK: do-catch
do {
    let data = try jsonEncoder.encode(model)
} catch {
    logger.error("Encoding failed: \(error)")
    throw error
}
```

### HIGH

```swift
// NG: guard let と if let の使い分けミス（早期リターンが必要な場面で if let を使い深いネスト）
func processUser(id: Int) {
    if let user = findUser(id: id) {
        if let profile = user.profile {
            if let email = profile.email {
                sendEmail(to: email) // ネストが深い
            }
        }
    }
}

// OK: guard let で早期リターン
func processUser(id: Int) {
    guard let user = findUser(id: id),
          let profile = user.profile,
          let email = profile.email else { return }
    sendEmail(to: email)
}
```

```swift
// NG: as! 強制ダウンキャスト
let vc = storyboard.instantiateViewController(withIdentifier: "Detail") as! DetailViewController
// 型が合わない時にクラッシュ

// OK: as? + nil チェック
guard let vc = storyboard.instantiateViewController(withIdentifier: "Detail") as? DetailViewController else {
    assertionFailure("DetailViewController が見つかりません")
    return
}
```

---

## カテゴリ3: Swift Concurrency（async/await）

### CRITICAL

```swift
// NG: actor でデータ競合を解決できるのに DispatchQueue で手動同期
class DataManager {
    private var cache: [String: Data] = [:]
    private let queue = DispatchQueue(label: "cache", attributes: .concurrent)

    func get(key: String) -> Data? {
        var result: Data?
        queue.sync { result = cache[key] }
        return result
    }

    func set(key: String, value: Data) {
        queue.async(flags: .barrier) { self.cache[key] = value }
    }
}

// OK: actor を使う
actor DataManager {
    private var cache: [String: Data] = [:]

    func get(key: String) -> Data? { cache[key] }
    func set(key: String, value: Data) { cache[key] = value }
}
```

```swift
// NG: @MainActor を付けるべき UI 更新コードが actor 外から呼ばれる
class NetworkClient {
    func fetchData() {
        URLSession.shared.dataTask(with: url) { data, _, _ in
            DispatchQueue.main.async { // 手動でメインスレッドに切り替え
                self.updateUI(data)
            }
        }.resume()
    }
}

// OK: @MainActor と async/await を使う
class NetworkClient {
    func fetchData() async throws {
        let (data, _) = try await URLSession.shared.data(from: url)
        await updateUI(data)
    }

    @MainActor
    func updateUI(_ data: Data) {
        // @MainActor によりメインスレッドで実行が保証される
    }
}
```

### HIGH

```swift
// NG: async let を使わず独立した async 処理を逐次 await
async func loadProfile(userId: String) async throws -> Profile {
    let user = try await fetchUser(userId)       // 完了を待ってから
    let posts = try await fetchPosts(userId)     // 開始（独立なのに直列）
    let friends = try await fetchFriends(userId) // 開始
    return Profile(user: user, posts: posts, friends: friends)
}

// OK: async let で並列実行
func loadProfile(userId: String) async throws -> Profile {
    async let user = fetchUser(userId)
    async let posts = fetchPosts(userId)
    async let friends = fetchFriends(userId)
    return try await Profile(user: user, posts: posts, friends: friends)
}
```

```swift
// NG: withCheckedContinuation を使わず callback を Task 内で呼ぶ（ハングの可能性）
func fetchLegacyData() async -> Data {
    var result: Data?
    legacyAPI.fetch { data in
        result = data
        // Task の completion を通知する手段がない
    }
    // result が設定される保証がないまま関数が終わる可能性
    return result ?? Data()
}

// OK: withCheckedContinuation でブリッジ
func fetchLegacyData() async -> Data {
    await withCheckedContinuation { continuation in
        legacyAPI.fetch { data in
            continuation.resume(returning: data)
        }
    }
}
```

```swift
// NG: Sendable conformance がなく actor 境界を越えてデータを渡す
class MutableData { // class は reference type → Sendable でない
    var value: Int = 0
}

actor DataActor {
    func process(_ data: MutableData) { // Sendable でない型を actor に渡す → 警告
        data.value = 42
    }
}

// OK: struct（value type）か Sendable を明示
struct ImmutableData: Sendable {
    let value: Int
}
```

---

## カテゴリ4: Swift 固有イディオム

### HIGH

```swift
// NG: struct + Optional フィールドで排他的状態を表現
struct NetworkState {
    var isLoading: Bool
    var data: Data?       // isLoading が false の時だけ有効
    var error: Error?     // エラーの時だけ有効
    // 複数の状態が同時に true になる不正状態が存在しうる
}

// OK: enum with associated values で排他的状態を表現
enum NetworkState {
    case loading
    case success(Data)
    case failure(Error)
}
```

```swift
// NG: Protocol extension の default 実装とクラス継承が混在して呼び出し規則が複雑
protocol Drawable {
    func draw()
}

extension Drawable {
    func draw() { print("Default draw") }
}

class Shape: Drawable {
    func draw() { print("Shape draw") }
}

class Circle: Shape {} // Circle().draw() は "Shape draw" か "Default draw" か？
// 正解: "Shape draw"（クラス継承が protocol extension より優先）
// → ポリモーフィズムが混乱しやすい設計
```

### MEDIUM

```swift
// NG: Equatable/Hashable の自動合成が利用できるのに手動実装
struct Point {
    let x: Double
    let y: Double

    // 全プロパティが Equatable なら自動合成可能なのに手動実装
    static func == (lhs: Point, rhs: Point) -> Bool {
        return lhs.x == rhs.x && lhs.y == rhs.y
    }
}

// OK: 自動合成を使う
struct Point: Equatable, Hashable {
    let x: Double
    let y: Double
    // == と hash(into:) は自動生成される
}
```

---

## カテゴリ5: エラーハンドリング

### HIGH

```swift
// NG: catch が catch all のみ（個別のエラーを識別できない）
do {
    try processData()
} catch {
    print("Error: \(error)") // どんなエラーかわからない
}

// OK: 具体的なエラー型を catch
do {
    try processData()
} catch NetworkError.timeout {
    retryLater()
} catch NetworkError.unauthorized {
    showLoginScreen()
} catch {
    reportUnexpectedError(error)
}
```

```swift
// NG: try? で全エラーを nil に潰す
let data = try? processData() // 何が失敗したか不明、エラーが握りつぶされる

// OK: do-catch でエラーを適切に処理
let data: Data
do {
    data = try processData()
} catch {
    logger.error("processData failed: \(error)")
    throw error
}
```

### MEDIUM

```swift
// NG: LocalizedError を準拠しない custom error
enum AppError: Error {
    case invalidInput
    case networkFailed
}
// エラーメッセージが "The operation couldn't be completed. (AppError error 0)." になる

// OK: LocalizedError を準拠
enum AppError: LocalizedError {
    case invalidInput
    case networkFailed

    var errorDescription: String? {
        switch self {
        case .invalidInput: return "入力値が不正です"
        case .networkFailed: return "ネットワークエラーが発生しました"
        }
    }
}
```

---

## カテゴリ6: パフォーマンス

### HIGH

```swift
// NG: メインスレッドで重い処理（UIハング）
class ImageProcessor {
    func applyFilter(to image: UIImage) -> UIImage {
        // 重い画像処理をメインスレッドで実行 → UIが固まる
        return heavyFilter(image)
    }
}

// OK: バックグラウンドスレッドで処理
func applyFilter(to image: UIImage) async -> UIImage {
    return await Task.detached(priority: .userInitiated) {
        self.heavyFilter(image)
    }.value
}
```

```swift
// NG: Array で頻繁な検索（O(n)）
var processedIds: [Int] = []

func isProcessed(id: Int) -> Bool {
    return processedIds.contains(id) // O(n)
}

// OK: Set を使う（O(1)）
var processedIds: Set<Int> = []

func isProcessed(id: Int) -> Bool {
    return processedIds.contains(id) // O(1)
}
```

---

## カテゴリ7: iOS/macOS 固有（UIKit/SwiftUI 検出時）

### CRITICAL

```swift
// NG: UserDefaults に大きなバイナリデータを保存（アプリ起動時の読み込みが遅くなる）
UserDefaults.standard.set(largeImageData, forKey: "profileImage") // NG: 数MBのデータをUserDefaultsに

// OK: ファイルシステムや Core Data を使う
let url = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
    .appendingPathComponent("profileImage.jpg")
try largeImageData.write(to: url)
```

### HIGH

```swift
// NG: @State と @StateObject / @ObservedObject の使い分けミス
struct ContentView: View {
    @ObservedObject var viewModel = ContentViewModel() // NG: @ObservedObject は外部から渡す用
    // → ContentView が再描画されるたびに新しい ViewModel が作成される可能性

    // OK: @StateObject を使う（このビューがオーナーの場合）
    @StateObject var viewModel = ContentViewModel()
}
```

```swift
// NG: Codable で CodingKeys を手動定義するが全フィールドに対応する key がない
struct User: Codable {
    let id: Int
    let name: String
    let createdAt: Date

    enum CodingKeys: String, CodingKey {
        case id
        case name
        // createdAt の CodingKey がない → DecodingError
    }
}

// OK: 全フィールドを含める
enum CodingKeys: String, CodingKey {
    case id
    case name
    case createdAt = "created_at"
}
```

```swift
// NG: NotificationCenter.addObserver で removeObserver を deinit で呼ばない
class MyViewController: UIViewController {
    override func viewDidLoad() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(handleNotification),
            name: .UIApplicationDidBecomeActive,
            object: nil
        )
        // removeObserver なし → observer が蓄積
    }
}

// OK: deinit で削除
deinit {
    NotificationCenter.default.removeObserver(self)
}
// または token ベースの API を使う
var token: NSObjectProtocol?

override func viewDidLoad() {
    token = NotificationCenter.default.addObserver(
        forName: .UIApplicationDidBecomeActive,
        object: nil,
        queue: .main
    ) { [weak self] _ in
        self?.handleNotification()
    }
}
deinit {
    if let token { NotificationCenter.default.removeObserver(token) }
}
```

### MEDIUM

```swift
// NG: DispatchQueue.main.async と @MainActor の混在
// モダンコードは @MainActor に統一
class ViewModel {
    func updateUI() {
        DispatchQueue.main.async { // 古いスタイル
            self.label.text = "Updated"
        }
    }
}

// OK: @MainActor を使う
@MainActor
class ViewModel {
    func updateUI() {
        label.text = "Updated" // @MainActor により自動的にメインスレッドで実行
    }
}
```
