# Python 専門コードレビュー

**品質基準**: CPython コア貢献者・PEP 著者レベル。
`pylint` / `mypy` / `ruff` をパスしても残る意味的バグ・ランタイム罠・設計問題を検出する。

**対象バージョン**: Python 3.8 以上（バージョン固有チェックは detector.md のマトリクスを参照）

---

## このファイルの使い方

Phase 1.5 担当エージェントは以下の12カテゴリを順に確認し、
検出した問題を `[CRITICAL/HIGH/MEDIUM/LOW]` で分類して報告する。
コード例・修正案・参照（PEP番号等）を必ず添付すること。

---

## カテゴリ1: Pythonic Idioms（PEP 20 準拠）

### CRITICAL

**C1-1: Mutable default argument**
```python
# NG
def add_item(item, lst=[]):
    lst.append(item)
    return lst

# OK
def add_item(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst
```
理由: デフォルト引数は関数定義時に一度だけ評価される。全呼び出しで同じオブジェクトを共有する。

**C1-2: Late binding closure（ループ内 lambda）**
```python
# NG
fns = [lambda: i for i in range(5)]
# fns[0]() → 4, fns[1]() → 4 (全て4を返す)

# OK
fns = [lambda x=i: x for i in range(5)]
```
理由: lambda はクロージャとして `i` の参照を保持し、ループ終了後の最終値を返す。

### HIGH

**H1-1: 同一性と等価性の混同**
```python
# NG: None, True, False はシングルトン → is を使う
if x == None: ...
if x == True: ...

# OK
if x is None: ...
if x: ...
```

**H1-2: range(len()) での列挙**
```python
# NG
for i in range(len(lst)):
    print(i, lst[i])

# OK
for i, v in enumerate(lst):
    print(i, v)
```

**H1-3: ループ内での文字列 `+` 結合（O(n²)）**
```python
# NG: 1万要素で顕著に遅くなる
result = ""
for s in strings:
    result += s

# OK
result = "".join(strings)
```

**H1-4: isinstance チェーンが 4 以上**
```python
# NG
if isinstance(x, A) or isinstance(x, B) or isinstance(x, C) or isinstance(x, D):
    ...

# OK (Python 3.10+)
match x:
    case A() | B() | C() | D():
        ...

# OK (全バージョン)
if isinstance(x, (A, B, C, D)):
    ...
```

### MEDIUM

**M1-1: ファルシー罠**
```python
# NG: 0, [], "" も False なので意図が不明確
value = value if value else default

# OK: None チェックが意図の場合
value = value if value is not None else default
```

**M1-2: 古い文字列フォーマット**
```python
# NG (Python 3.6+)
msg = "Hello, %s" % name
msg = "Hello, {}".format(name)

# OK
msg = f"Hello, {name}"
```

---

## カテゴリ2: 型ヒント（PEP 484/526/544/585/604）

### CRITICAL

**C2-1: `from __future__ import annotations` 漏れ（Python < 3.10）**
```python
# NG (Python 3.9): X | Y は 3.10+ の機能
def f(x: int | None) -> str | int:  # SyntaxError on 3.9

# OK (Python 3.9)
from __future__ import annotations
def f(x: int | None) -> str | int:  # 文字列として評価され動作する
```

### HIGH

**H2-1: `Any` を公開 API の戻り値型に使用**
```python
# NG: 呼び出し側で型情報が失われる
from typing import Any
def get_value(key: str) -> Any: ...

# OK
def get_value(key: str) -> int | str | None: ...
```

**H2-2: Python 3.9+ で旧 typing 表記を使用**
```python
# NG (Python 3.9+)
from typing import Dict, List, Optional, Tuple
def f(x: Optional[str]) -> Dict[str, List[int]]: ...

# OK
def f(x: str | None) -> dict[str, list[int]]: ...
```

**H2-3: `TypeVar` に制約なし**
```python
# NG: T は実質 Any
T = TypeVar('T')
def first(lst: list[T]) -> T: ...

# OK: 上限境界を指定
T = TypeVar('T', bound=Comparable)
```

**H2-4: `Protocol` を isinstance で使うが `@runtime_checkable` なし**
```python
# NG
class Drawable(Protocol):
    def draw(self) -> None: ...

isinstance(obj, Drawable)  # TypeError: not @runtime_checkable

# OK
@runtime_checkable
class Drawable(Protocol):
    def draw(self) -> None: ...
```

**H2-5: `@overload` 未使用で複数の引数型パターンを Union で誤魔化す**
```python
# NG
def process(x: int | str) -> int | str:  # 戻り値型が曖昧

# OK: int を渡すと int が返ることを型で表現
@overload
def process(x: int) -> int: ...
@overload
def process(x: str) -> str: ...
def process(x):
    return x
```

### MEDIUM

**M2-1: `ClassVar` と `Final` の使い分け漏れ**
```python
from typing import ClassVar, Final

class Config:
    # NG: インスタンス変数と区別できない
    MAX_RETRIES = 3

    # OK: クラス変数
    MAX_RETRIES: ClassVar[int] = 3
    # OK: 変更不可定数
    VERSION: Final = "1.0.0"
```

**M2-2: `TypedDict` の Required/NotRequired 未使用**
```python
# NG
class Config(TypedDict, total=False):
    host: str          # オプション
    port: int          # オプション
    debug: bool        # オプション（でも必須にしたい）

# OK (Python 3.11+)
class Config(TypedDict):
    host: str          # 必須
    port: int          # 必須
    debug: NotRequired[bool]  # オプション
```

---

## カテゴリ3: 例外処理

### CRITICAL

**C3-1: bare `except:`**
```python
# NG: SystemExit, KeyboardInterrupt, GeneratorExit も捕捉する
try:
    risky()
except:
    pass

# OK
try:
    risky()
except Exception:
    pass
```

**C3-2: Exception chain 消滅**
```python
# NG: 元の例外情報が失われる
try:
    parse(data)
except ValueError:
    raise RuntimeError("parsing failed")  # from なし

# OK
try:
    parse(data)
except ValueError as e:
    raise RuntimeError("parsing failed") from e
```

### HIGH

**H3-1: `try` ブロックのスコープが広すぎる**
```python
# NG: 20行の try ブロック → どの行で例外が起きるか不明
try:
    a = fetch_user(id)
    b = calculate(a)
    c = save(b)
    d = notify(c)
    ...20行...
except ValueError:
    handle()

# OK: 個別に捕捉
try:
    a = fetch_user(id)
except ValueError as e:
    raise UserNotFoundError(id) from e
```

**H3-2: `assert` による入力検証**
```python
# NG: python -O で assert は無効化される
def divide(a, b):
    assert b != 0, "b must not be zero"
    return a / b

# OK
def divide(a, b):
    if b == 0:
        raise ValueError("b must not be zero")
    return a / b
```

**H3-3: `finally` ブロックで例外を raise**
```python
# NG: 元の例外を上書きする
try:
    risky()
except Exception as e:
    original = e
finally:
    cleanup()  # ここで例外が起きると original が消える
    raise AnotherError()  # 元の例外が消滅
```

### MEDIUM

**M3-1: 例外メッセージに原因値なし**
```python
# NG
raise ValueError("invalid value")

# OK: デバッグ可能なメッセージ
raise ValueError(f"invalid value: {value!r} (expected positive integer)")
```

---

## カテゴリ4: メモリ管理

### CRITICAL

**C4-1: 大きなオブジェクトの循環参照 + weakref なし**
```python
# NG: parent と child が互いに参照 → CPython の参照カウントで回収されない
class Node:
    def __init__(self, parent=None):
        self.parent = parent  # 強参照
        self.children = []

# OK: 親への参照を弱参照に
import weakref

class Node:
    def __init__(self, parent=None):
        self._parent = weakref.ref(parent) if parent else None

    @property
    def parent(self):
        return self._parent() if self._parent else None
```

### HIGH

**H4-1: `functools.lru_cache` をインスタンスメソッドに適用**
```python
# NG: self が強参照でキャッシュに保持 → インスタンスが GC されない
class Service:
    @functools.lru_cache(maxsize=128)
    def get_data(self, key):
        return expensive_fetch(key)

# OK: methodtools.lru_cache か、クラス外のキャッシュを使う
from methodtools import lru_cache

class Service:
    @lru_cache(maxsize=128)
    def get_data(self, key):
        return expensive_fetch(key)
```

**H4-2: `__slots__` 未使用の大量インスタンス生成**
```python
# NG: 各インスタンスが __dict__ を持つ（40-50% ヒープ超過）
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

# OK: 100万インスタンス生成するなら必須
class Point:
    __slots__ = ('x', 'y')
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

**H4-3: 大ファイルの一括ロード**
```python
# NG: 10GBファイルで OOM
content = open("huge.log").read()

# OK: ジェネレータでストリーミング
def read_lines(path):
    with open(path) as f:
        for line in f:
            yield line.rstrip()
```

---

## カテゴリ5: 並行処理・非同期

### CRITICAL

**C5-1: async 関数内でブロッキング呼び出し**
```python
# NG: イベントループをブロックする
async def fetch_user(user_id):
    time.sleep(1)  # ブロッキング!
    response = requests.get(url)  # ブロッキング!
    return response.json()

# OK
import asyncio
import httpx

async def fetch_user(user_id):
    await asyncio.sleep(1)
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
    return response.json()
```

**C5-2: CPU バウンドに threading.Thread**
```python
# NG: GIL により CPU バウンドは並列化されない
threads = [threading.Thread(target=cpu_intensive, args=(x,)) for x in data]

# OK
from concurrent.futures import ProcessPoolExecutor
with ProcessPoolExecutor() as executor:
    results = list(executor.map(cpu_intensive, data))
```

**C5-3: ネスト async 環境で asyncio.run()**
```python
# NG: Jupyter / FastAPI 等の既存イベントループ内で呼ぶ
asyncio.run(main())  # RuntimeError: This event loop is already running

# OK
import nest_asyncio
nest_asyncio.apply()  # Jupyter 用
# または
await main()  # すでに async 環境なら直接 await
```

### HIGH

**H5-1: asyncio.create_task() の結果を破棄**
```python
# NG: タスクが GC に回収される可能性がある
asyncio.create_task(background_work())

# OK: 参照を保持する
background_tasks = set()
task = asyncio.create_task(background_work())
background_tasks.add(task)
task.add_done_callback(background_tasks.discard)
```

**H5-2: asyncio.gather のエラーハンドリングなし**
```python
# NG: 1つが失敗すると他もキャンセルされる
results = await asyncio.gather(task1(), task2(), task3())

# OK: 全結果を収集（失敗も含む）
results = await asyncio.gather(task1(), task2(), task3(), return_exceptions=True)
for r in results:
    if isinstance(r, Exception):
        handle_error(r)
```

**H5-3: タイムアウトなしの無限待機**
```python
# NG: ネットワーク遅延で永久ブロック
result = await slow_operation()

# OK
try:
    result = await asyncio.wait_for(slow_operation(), timeout=30.0)
except asyncio.TimeoutError:
    handle_timeout()
```

---

## カテゴリ6: セキュリティ（Python 固有）

### CRITICAL

**C6-1: pickle.loads でユーザーデータを逆シリアライズ**
```python
# NG: 任意コード実行 (RCE)
data = pickle.loads(user_provided_bytes)

# OK: JSON や msgpack を使う
import json
data = json.loads(user_provided_json)
```

**C6-2: yaml.load に SafeLoader なし**
```python
# NG: !! python/object: 等で RCE
data = yaml.load(user_yaml)

# OK
data = yaml.safe_load(user_yaml)
# または
data = yaml.load(user_yaml, Loader=yaml.SafeLoader)
```

**C6-3: eval/exec にユーザー入力**
```python
# NG
result = eval(user_expression)

# OK: ast.literal_eval（リテラルのみ許可）
import ast
result = ast.literal_eval(user_expression)
```

**C6-4: subprocess shell=True にユーザー入力**
```python
# NG: コマンドインジェクション
subprocess.call(f"ls {user_path}", shell=True)

# OK: リストでコマンドを渡す（shell=False）
subprocess.call(["ls", user_path])
```

**C6-5: tempfile.mktemp() → TOCTOU 競合**
```python
# NG: チェックと使用の間に別プロセスが介入できる
path = tempfile.mktemp()
open(path, 'w').write(data)  # TOCTOU

# OK
fd, path = tempfile.mkstemp()
with os.fdopen(fd, 'w') as f:
    f.write(data)
```

### HIGH

**H6-1: MD5/SHA1 でパスワードハッシュ**
```python
# NG
hashed = hashlib.md5(password.encode()).hexdigest()

# OK
import bcrypt
hashed = bcrypt.hashpw(password.encode(), bcrypt.gensalt())
```

**H6-2: random モジュールでセキュリティトークン生成**
```python
# NG: 予測可能
token = ''.join(random.choices(string.ascii_letters, k=32))

# OK
import secrets
token = secrets.token_hex(32)
```

**H6-3: 未信頼 XML のパース（XXE 攻撃）**
```python
# NG: expat は外部エンティティを展開する
import xml.etree.ElementTree as ET
tree = ET.parse(user_xml)

# OK
import defusedxml.ElementTree as ET
tree = ET.parse(user_xml)
```

---

## カテゴリ7: dunder メソッドの正確性

### CRITICAL

**C7-1: `__eq__` を定義して `__hash__` を定義しない**
```python
# NG: Python 3 は __eq__ 定義時に __hash__ を None に設定
class Point:
    def __eq__(self, other):
        return self.x == other.x and self.y == other.y
    # __hash__ がない → dict/set に使えない

# OK
class Point:
    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

    def __hash__(self):
        return hash((self.x, self.y))
```

### HIGH

**H7-1: `__enter__` と `__exit__` の片方のみ定義**
```python
# NG: with ステートメントが壊れる
class Resource:
    def __enter__(self):
        self.acquire()
        return self
    # __exit__ がない → with 使用時に AttributeError

# OK
class Resource:
    def __enter__(self):
        self.acquire()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.release()
        return False  # 例外を再 raise
```

**H7-2: `__iter__` を定義して iterable と iterator を混同**
```python
# NG: イテレータは __iter__ と __next__ の両方が必要
class Counter:
    def __iter__(self):  # __next__ がない → for ループで TypeError
        ...

# OK: iterator（一度だけ使用可能）
class Counter:
    def __iter__(self):
        return self

    def __next__(self):
        ...

# OK: iterable（何度でも使用可能）
class CounterIterable:
    def __iter__(self):
        return CounterIterator(self)
```

---

## カテゴリ8: デスクリプタ・クラス設計

### HIGH

**H8-1: `@property` getter に副作用**
```python
# NG: 呼ぶたびに DB クエリ → パフォーマンス問題に気づきにくい
class User:
    @property
    def orders(self):
        return db.query(Order).filter_by(user_id=self.id).all()  # 毎回クエリ

# OK: メソッドに変更（副作用があることを明示）
class User:
    def get_orders(self):
        return db.query(Order).filter_by(user_id=self.id).all()
```

**H8-2: `@classmethod` と `@staticmethod` の誤使用**
```python
# NG: self が使われないが @classmethod を使っている
class Parser:
    @classmethod
    def validate(cls, data):
        return len(data) > 0  # cls を使っていない → @staticmethod が正しい

# OK
class Parser:
    @staticmethod
    def validate(data):
        return len(data) > 0
```

### MEDIUM

**M8-1: `Protocol` と `ABC` の選択ミス**
```python
# NG: ABC は継承が強制される
class Drawable(ABC):
    @abstractmethod
    def draw(self): ...

# OK: Protocol は継承不要（structural subtyping）
class Drawable(Protocol):
    def draw(self) -> None: ...
# → draw() を持つ任意のクラスが Drawable として扱われる
```

**M8-2: `dataclass(frozen=True)` 未使用の不変オブジェクト**
```python
# NG: 変更されることを想定していない DTO が mutable
@dataclass
class Point:
    x: float
    y: float

# OK
@dataclass(frozen=True)
class Point:
    x: float
    y: float
```

---

## カテゴリ9: テスト品質

### HIGH

**H9-1: pytest.fixture の scope 未指定**
```python
# NG: 重い DB セットアップが毎テスト実行される（デフォルト scope="function"）
@pytest.fixture
def db_connection():
    conn = create_heavy_db_connection()
    yield conn
    conn.close()

# OK: セッション全体で1回だけ
@pytest.fixture(scope="session")
def db_connection():
    conn = create_heavy_db_connection()
    yield conn
    conn.close()
```

**H9-2: `@pytest.mark.parametrize` 未使用**
```python
# NG: 同じロジックの繰り返し
def test_positive():
    assert is_valid(1)

def test_negative():
    assert not is_valid(-1)

def test_zero():
    assert not is_valid(0)

# OK
@pytest.mark.parametrize("value,expected", [
    (1, True),
    (-1, False),
    (0, False),
    (100, True),
])
def test_is_valid(value, expected):
    assert is_valid(value) == expected
```

**H9-3: 時刻依存テスト（datetime.now() をモックしない）**
```python
# NG: 実行日時によって結果が変わる
def test_is_expired():
    token = Token(created_at=datetime(2020, 1, 1))
    assert token.is_expired()  # 2020年以降は常に True だが...

# OK
def test_is_expired(freezer):
    freezer.move_to("2020-06-01")
    token = Token(created_at=datetime(2020, 1, 1))
    assert token.is_expired()
```

---

## カテゴリ10: パッケージング・インポート

### CRITICAL

**C10-1: 本番コードに sys.path.append()**
```python
# NG: 順序依存・環境依存で壊れやすい
import sys
sys.path.append('/home/user/mylib')
from mylib import utils

# OK: pyproject.toml で依存関係を管理、パッケージとしてインストール
```

### HIGH

**H10-1: モジュール間の循環 import**
```python
# NG: a.py が b.py を import し、b.py が a.py を import
# → ImportError または部分初期化オブジェクトが渡される

# OK: 共通部分を c.py に切り出す、または遅延 import
def get_b():
    from . import b  # 関数内で遅延 import
    return b.something
```

**H10-2: `setup.py` のみ（pyproject.toml 未移行）**
```
# NG: setup.py は PEP 517/518 以前の旧方式
python setup.py install  # 非推奨

# OK: pyproject.toml に移行
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "mypackage"
version = "1.0.0"
```

---

## カテゴリ11: よくある罠（Gotchas）

### CRITICAL

**C11-1: list.sort() の戻り値を使用**
```python
# NG: sort() は None を返す（in-place ソート）
sorted_list = my_list.sort()  # None!
print(sorted_list[0])  # TypeError

# OK
my_list.sort()        # in-place
sorted_list = sorted(my_list)  # 新しいリストを返す
```

### HIGH

**H11-1: `is` での値比較**
```python
# NG: CPython の整数インターン（-5〜256）に依存した偶然の動作
x = 256
x is 256  # True (CPython では True だが保証なし)
x = 257
x is 257  # False!

# OK
x == 257  # 常に正しい
```

**H11-2: イテレート中のリスト変更**
```python
# NG: 偶数番目の要素がスキップされる
lst = [1, 2, 3, 4, 5]
for item in lst:
    if item % 2 == 0:
        lst.remove(item)  # NG: イテレート中に変更

# OK: コピーかリスト内包表記を使う
lst = [item for item in lst if item % 2 != 0]
```

**H11-3: NaN の比較**
```python
# NG: NaN は自分自身と等しくない
value = float('nan')
if value == float('nan'):  # 常に False
    handle_nan()

# OK
import math
if math.isnan(value):
    handle_nan()
```

---

## カテゴリ12: フレームワーク固有（条件付き）

### Django（`django` が dependencies に存在する場合）

**D1: N+1 クエリ**
```python
# NG: N+1（Userが100人いれば100回クエリ）
users = User.objects.all()
for user in users:
    print(user.profile.bio)  # 毎回 profile を取得

# OK
users = User.objects.select_related('profile').all()
```

**D2: DEBUG=True が環境変数で保護されていない**
```python
# NG
DEBUG = True  # ハードコード

# OK
import os
DEBUG = os.environ.get('DJANGO_DEBUG', 'False') == 'True'
```

**D3: raw() クエリに未パラメータ化**
```python
# NG
User.objects.raw(f"SELECT * FROM users WHERE name = '{name}'")

# OK
User.objects.raw("SELECT * FROM users WHERE name = %s", [name])
```

### FastAPI（`fastapi` が dependencies に存在する場合）

**F1: response_model なし → 内部フィールド漏洩**
```python
# NG: password ハッシュ等が漏洩する可能性
@app.get("/users/{id}")
async def get_user(id: int):
    return db.get_user(id)  # 全フィールドが返る

# OK
@app.get("/users/{id}", response_model=UserPublic)
async def get_user(id: int):
    return db.get_user(id)
```

**F2: エンドポイント内のブロッキング処理**
```python
# NG
@app.get("/data")
async def get_data():
    result = requests.get(external_url)  # ブロッキング

# OK
@app.get("/data")
async def get_data():
    async with httpx.AsyncClient() as client:
        result = await client.get(external_url)
```

### SQLAlchemy（`sqlalchemy` が dependencies に存在する場合）

**S1: Session が finally/context manager 外**
```python
# NG
session = Session()
result = session.query(User).all()
# session.close() が漏れる

# OK
with Session() as session:
    result = session.query(User).all()
```

**S2: text() に未パラメータ化**
```python
# NG
session.execute(text(f"SELECT * FROM users WHERE id = {user_id}"))

# OK
session.execute(text("SELECT * FROM users WHERE id = :id"), {"id": user_id})
```
