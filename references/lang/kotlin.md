# Kotlin 言語固有ディープレビュー

**品質基準**: JetBrains Kotlin チーム・Google Android エンジニアレベル。
ktlint / detekt をパスしても残る問題を検出する。

---

## カテゴリ1: Null 安全の深層

### CRITICAL

```kotlin
// NG: !! (non-null assertion) を production コードに多用
val name = user!!.profile!!.name!! // null が来たら NPE クラッシュ

// OK: safe call + let/run/elvis を使う
val name = user?.profile?.name ?: "Unknown"
```

### HIGH

```kotlin
// NG: Java との相互運用で @Nullable アノテーションを無視した型推論
// Java: @Nullable String getName() { ... }
val name: String = javaClass.name // Kotlin はプラットフォーム型 String! として推論
// null が来た場合に NullPointerException

// OK: nullable 型として扱う
val name: String? = javaClass.name
val display = name ?: "Unknown"
```

```kotlin
// NG: lateinit var を isInitialized チェックなしで使用
lateinit var adapter: RecyclerView.Adapter<*>

fun onResume() {
    adapter.notifyDataSetChanged() // setup() 前に呼ばれると UninitializedPropertyAccessException
}

// OK: isInitialized でチェック
if (::adapter.isInitialized) {
    adapter.notifyDataSetChanged()
}
```

### MEDIUM

```kotlin
// NG: lateinit vs lazy の選択ミス
lateinit var config: Config // val にできるのに var にしている

// OK: lazy を使う（thread-safe、val で定義できる）
val config: Config by lazy {
    loadConfig()
}
```

---

## カテゴリ2: Kotlin イディオム

### HIGH

```kotlin
// NG: var を使えるが変更されない変数
var count = items.size // 変更されない
var message = "Hello, $name"

// OK: val を使う
val count = items.size
val message = "Hello, $name"
```

```kotlin
// NG: data holder クラスに data class を使わない
class UserDto(
    val id: Long,
    val name: String,
    val email: String
) // equals/hashCode/copy/toString が手実装になる

// OK: data class
data class UserDto(
    val id: Long,
    val name: String,
    val email: String
)
```

```kotlin
// NG: singleton に companion object や static 変数を使う
class Database {
    companion object {
        private var instance: Database? = null
        fun getInstance(): Database = instance ?: Database().also { instance = it }
    }
}

// OK: object を使う
object Database {
    fun query(sql: String): Result { ... }
}
```

```kotlin
// NG: sealed class の when に else ブランチが必要な設計（sealed の意味がない）
sealed class Result
class Success(val data: String) : Result()
class Failure(val error: Error) : Result()
open class Pending : Result() // open にすると外部から継承できてしまう

fun handle(result: Result) = when (result) {
    is Success -> ...
    is Failure -> ...
    else -> ... // else が必要 → sealed の恩恵を失う
}

// OK: sealed class の全 subclass を網羅し else 不要にする
```

---

## カテゴリ3: コルーチン・非同期

### CRITICAL

```kotlin
// NG: GlobalScope.launch（structured concurrency 違反）
GlobalScope.launch {
    fetchData() // ライフサイクル管理不可、テストが困難
}

// OK: viewModelScope / lifecycleScope / coroutineScope を使う
viewModelScope.launch {
    fetchData()
}
```

```kotlin
// NG: runBlocking をコルーチン内から呼ぶ（デッドロードのリスク）
suspend fun processData() {
    val result = runBlocking { // suspend 関数内の runBlocking はデッドロックの可能性
        async { fetchFromDb() }.await()
    }
}

// OK: runBlocking を使わず suspend で
suspend fun processData() {
    val result = fetchFromDb()
}
```

### HIGH

```kotlin
// NG: Dispatchers.IO を CPU バウンド処理に使用
launch(Dispatchers.IO) {
    val result = heavyComputation() // CPU バウンドに IO dispatcher は非効率
}

// OK: Dispatchers.Default を使う
launch(Dispatchers.Default) {
    val result = heavyComputation()
}
```

```kotlin
// NG: async の結果を await せずに破棄（例外が無視される）
launch {
    val deferred = async { fetchData() } // deferred を無視
    // await() が呼ばれないと例外が飲み込まれる
    doOtherWork()
}

// OK: await() を必ず呼ぶ
launch {
    val result = async { fetchData() }.await()
    process(result)
}
```

```kotlin
// NG: launch の例外を try/catch で捕まえようとする
launch {
    try {
        riskyOperation()
    } catch (e: Exception) {
        // これは launch の外の例外をキャッチしない
    }
}

// OK: CoroutineExceptionHandler を使う
val handler = CoroutineExceptionHandler { _, exception ->
    Log.e("TAG", "Coroutine exception: $exception")
}
launch(handler) {
    riskyOperation()
}
```

```kotlin
// NG: Flow の収集を lifecycle スコープ外で行う（memory leak / crash）
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // NG: lifecycleScope を使わない
        GlobalScope.launch {
            viewModel.data.collect { updateUI(it) } // Fragment が destroy されても続く
        }
    }
}

// OK: repeatOnLifecycle を使う
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    viewLifecycleOwner.lifecycleScope.launch {
        viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
            viewModel.data.collect { updateUI(it) }
        }
    }
}
```

---

## カテゴリ4: 関数型・高階関数

### HIGH

```kotlin
// NG: scope functions を3重以上にネスト（可読性崩壊）
val result = user?.let { u ->
    u.profile?.run {
        address?.also { addr ->
            log.debug("Address: $addr") // 3重ネスト、どの this か不明確
        }
    }
}

// OK: let は1-2段まで、複雑なら通常の関数に分解
```

```kotlin
// NG: inline 関数で reified が使えるのに使っていない
inline fun <T> parseJson(json: String, clazz: Class<T>): T { // Class を引数で渡す必要がある
    return objectMapper.readValue(json, clazz)
}
parseJson(json, User::class.java)

// OK: reified で型情報を使う
inline fun <reified T> parseJson(json: String): T {
    return objectMapper.readValue(json, T::class.java)
}
parseJson<User>(json)
```

---

## カテゴリ5: 型システム・Generics

### HIGH

```kotlin
// NG: in/out variance が正しくない generic クラス
class Box<T>(var value: T) // invariant: Box<String> を Box<Any> に代入できない

// 読み取り専用なら covariant
class Box<out T>(val value: T) // Box<String> を Box<Any> に代入できる

// 書き込み専用なら contravariant
class Processor<in T> { fun process(value: T) { ... } }
```

```kotlin
// NG: Java 相互運用で @JvmStatic, @JvmField, @JvmOverloads が漏れている
object Config {
    val DEFAULT_TIMEOUT = 30 // Java から Config.INSTANCE.getDEFAULT_TIMEOUT() になる
    fun create() = Config() // Java から Config.INSTANCE.create() になる
}

// OK: アノテーションを付ける
object Config {
    @JvmField val DEFAULT_TIMEOUT = 30 // Java から Config.DEFAULT_TIMEOUT
    @JvmStatic fun create() = Config() // Java から Config.create()
}
```

```kotlin
// NG: crossinline/noinline ラムダパラメータの指定漏れ
inline fun execute(block: () -> Unit) {
    val runnable = Runnable { block() } // NG: inline 関数内で他の lambda に渡す場合は crossinline が必要
    thread { runnable.run() }
}

// OK
inline fun execute(crossinline block: () -> Unit) {
    val runnable = Runnable { block() }
    thread { runnable.run() }
}
```

---

## カテゴリ6: Android 固有（android 依存検出時）

### CRITICAL

```kotlin
// NG: Activity/Fragment/View への参照を coroutine や callback で保持（memory leak）
class MyActivity : AppCompatActivity() {
    private var job: Job? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        job = GlobalScope.launch {
            val data = fetchData()
            // NG: Activity が destroy されても this@MyActivity を参照
            textView.text = data
        }
    }
}

// OK: lifecycleScope を使う（Activity の破棄とともにキャンセルされる）
override fun onCreate(savedInstanceState: Bundle?) {
    lifecycleScope.launch {
        val data = fetchData()
        textView.text = data
    }
}
```

### HIGH

```kotlin
// NG: ViewModel で LiveData を使うが StateFlow の方が適切（Kotlin 環境）
class MyViewModel : ViewModel() {
    private val _state = MutableLiveData<UiState>()
    val state: LiveData<UiState> = _state
}

// OK: StateFlow（初期値が必須で型安全、Kotlin Coroutines と統合しやすい）
class MyViewModel : ViewModel() {
    private val _state = MutableStateFlow(UiState.Loading)
    val state: StateFlow<UiState> = _state.asStateFlow()
}
```

```kotlin
// NG: viewModelScope と lifecycleScope の使い分けミス
class MyViewModel : ViewModel() {
    fun loadData() {
        lifecycleScope.launch { // NG: ViewModel で lifecycleScope は使えない（そもそもコンパイルエラー）
            // viewModelScope を使うべき
        }
    }
}

// Fragment/Activity では
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // OK: UI の更新なら lifecycleScope
        viewLifecycleOwner.lifecycleScope.launch { ... }
        // NG: ViewModel の処理を Fragment の lifecycleScope で実行すると
        //     Fragment が消えた時にキャンセルされてしまう
    }
}
```

---

## カテゴリ7: テスト・品質

### HIGH

```kotlin
// NG: runBlocking を使ったコルーチンテスト（TestCoroutineDispatcher を使えない）
@Test
fun testFetchData() = runBlocking {
    val result = viewModel.fetchData()
    assertEquals(expected, result)
}

// OK: runTest（kotlinx-coroutines-test）を使う
@Test
fun testFetchData() = runTest {
    val result = viewModel.fetchData()
    assertEquals(expected, result)
}
```

```kotlin
// NG: TestCoroutineDispatcher が deprecated（kotlinx-coroutines-test 1.6+）
val testDispatcher = TestCoroutineDispatcher() // Deprecated

// OK: UnconfinedTestDispatcher か StandardTestDispatcher
val testDispatcher = UnconfinedTestDispatcher()
// または遅延実行のテスト
val testDispatcher = StandardTestDispatcher()
```

```kotlin
// NG: mockk の mock をテスト間で状態リセットしない
val userRepository = mockk<UserRepository>()

@BeforeEach
fun setup() {
    // clearMocks なし → 前のテストの verify が残ってしまう
}

// OK
@BeforeEach
fun setup() {
    clearMocks(userRepository)
    // または
}
@AfterEach
fun teardown() {
    unmockkAll()
}
```

### MEDIUM

```kotlin
// NG: delay() をスキップするために手動で時間を進めない
@Test
fun testDelayedOperation() = runTest {
    launch {
        delay(1000)
        flag = true
    }
    // delay(1000) を実際に待つ → テストが遅い
    delay(1100)
    assertTrue(flag)
}

// OK: advanceUntilIdle() で時間をスキップ
@Test
fun testDelayedOperation() = runTest {
    launch {
        delay(1000)
        flag = true
    }
    advanceUntilIdle() // 全ての保留中の coroutine を即座に実行
    assertTrue(flag)
}
```
