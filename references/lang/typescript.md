# TypeScript / JavaScript 言語固有ディープレビュー

**品質基準**: TC39 委員 + TypeScript コンパイラチームレベル。
`tsc --strict`, `eslint` をパスしても残る問題を検出する。

---

## カテゴリ1: TypeScript 型システムの深層

### CRITICAL

```typescript
// NG: @ts-ignore にコメントなし
// @ts-ignore
const value = getValue() as any;

// OK: 理由を必ず説明
// @ts-ignore: ライブラリの型定義が不完全で型推論できないため（XXX v1.2.3 issue #456）
const value = getValue() as any;
```

```typescript
// NG: @ts-nocheck をファイルレベルで使用
// @ts-nocheck  ← 型チェックを完全に無効化

// OK: 型定義を修正するか、最小限の @ts-ignore を使う
```

```typescript
// NG: double assertion（型チェックの完全回避）
const value = (input as unknown) as TargetType;

// OK: 型ガードや適切なキャストを使う
function isTargetType(v: unknown): v is TargetType {
    return typeof v === 'object' && v !== null && 'field' in v;
}
```

### HIGH

```typescript
// NG: branded types 未使用（意味的に異なる値が同じ型）
type UserId = string;
type Email = string;
function sendEmail(userId: UserId, email: Email) { ... }
sendEmail(email, userId); // 引数を逆に渡してもコンパイラが検出しない

// OK: branded types で型安全
type UserId = string & { readonly _brand: 'UserId' };
type Email = string & { readonly _brand: 'Email' };
function createUserId(s: string): UserId { return s as UserId; }
```

```typescript
// NG: discriminated union を使わず手動 narrowing
type Shape = {
    type: string;
    radius?: number;
    width?: number;
    height?: number;
};
// type プロパティが string だと絞り込みが型安全でない

// OK: discriminated union
type Shape =
    | { type: 'circle'; radius: number }
    | { type: 'rectangle'; width: number; height: number };

function area(shape: Shape): number {
    switch (shape.type) {
        case 'circle': return Math.PI * shape.radius ** 2;
        case 'rectangle': return shape.width * shape.height;
    }
}
```

```typescript
// NG: satisfies 演算子を使わず as const で型推論を壊す
const config = {
    port: 3000,
    host: 'localhost',
} as const; // Record<string, unknown> として扱えなくなる

// OK: satisfies でより精度の高い推論
const config = {
    port: 3000,
    host: 'localhost',
} satisfies ServerConfig; // 型チェックしつつ元の型を維持
```

```typescript
// NG: 3段以上ネストした conditional type（可読性崩壊）
type DeepNested<T> = T extends string
    ? T extends `${infer A}_${infer B}`
        ? A extends keyof SomeMap
            ? SomeMap[A]
            : never
        : never
    : never;

// OK: ヘルパー型に分割
type ParseKey<T extends string> = T extends `${infer A}_${infer B}` ? A : never;
type ResolveKey<K extends string> = K extends keyof SomeMap ? SomeMap[K] : never;
type DeepNested<T extends string> = ResolveKey<ParseKey<T>>;
```

---

## カテゴリ2: tsconfig.json の厳格性

### HIGH

```json
// NG: strict: true 未設定
{
  "compilerOptions": {
    "target": "ES2020"
    // strict なし → noImplicitAny, strictNullChecks 等が全て無効
  }
}

// OK
{
  "compilerOptions": {
    "target": "ES2020",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true
  }
}
```

```json
// NG: noUncheckedIndexedAccess が false（デフォルト）
// const arr: string[] = ['a', 'b'];
// const item = arr[999]; // item の型が string になり undefined チェックなし

// OK: noUncheckedIndexedAccess: true にすると item の型が string | undefined になる
```

---

## カテゴリ3: ランタイムトラップ（JavaScript 仕様レベル）

### CRITICAL

```typescript
// NG: typeof null === "object" の判定漏れ
function isObject(v: unknown): boolean {
    return typeof v === 'object'; // null も通る
}

// OK: null チェックを先に
function isObject(v: unknown): v is object {
    return v !== null && typeof v === 'object';
}
```

### HIGH

```typescript
// NG: NaN との比較（常に false）
if (value === NaN) { ... } // 絶対に true にならない

// OK
if (Number.isNaN(value)) { ... }
```

```typescript
// NG: == による型強制比較（予期しない true）
"0" == false    // true（型強制）
[] == false     // true（型強制）
null == undefined // true

// OK: === を使う（TypeScript の strict では警告が出るがコードに残ることがある）
```

```typescript
// NG: 浮動小数点の等値比較
if (0.1 + 0.2 === 0.3) { ... } // false！

// OK: epsilon 比較
const EPSILON = Number.EPSILON;
if (Math.abs(0.1 + 0.2 - 0.3) < EPSILON) { ... }
```

```typescript
// NG: parseInt に radix なし（古い環境での 8 進数解釈）
parseInt("08") // 一部の環境で 0 を返す

// OK: 常に radix を指定
parseInt("08", 10) // 8
```

### MEDIUM

```typescript
// NG: Object.keys() が string[] を返し keyof T と型が合わない
const obj = { a: 1, b: 2 };
for (const key of Object.keys(obj)) {
    obj[key]; // NG: key が string なので TS エラー
}

// OK: 型アサーションか Object.entries を使う
for (const [key, value] of Object.entries(obj)) { ... }
// または
(Object.keys(obj) as Array<keyof typeof obj>).forEach(key => obj[key]);
```

```typescript
// NG: for...in がプロトタイプチェーンのプロパティも列挙
for (const key in obj) {
    // key にはプロトタイプのプロパティも含まれることがある
}

// OK: hasOwnProperty チェックか for...of Object.keys()
for (const key in obj) {
    if (Object.prototype.hasOwnProperty.call(obj, key)) { ... }
}
```

---

## カテゴリ4: Async/Promise パターン

### CRITICAL

```typescript
// NG: async 関数内に await が1つもない
async function getData(): Promise<Data> {
    return fetchSync(); // async キーワードが不要か処理が不完全
}

// OK: 非同期処理が必要な場合のみ async を使う
function getData(): Data {
    return fetchSync();
}
// または非同期処理を追加
async function getData(): Promise<Data> {
    return await fetchAsync();
}
```

### HIGH

```typescript
// NG: 独立した fetch を直列 await（レイテンシが加算される）
const user = await fetchUser(id);
const posts = await fetchPosts(id); // user の完了を待ってから開始（独立しているのに）

// OK: Promise.all で並列実行
const [user, posts] = await Promise.all([fetchUser(id), fetchPosts(id)]);
```

```typescript
// NG: Promise.all vs Promise.allSettled の選択ミス
// Promise.all: 1つ失敗で全失敗
// Promise.allSettled: 各々の結果を返す（全て完了するまで待つ）

// 独立したタスクで部分的な失敗を許容する場合
const results = await Promise.allSettled([task1(), task2(), task3()]);
const failures = results.filter(r => r.status === 'rejected');
```

```typescript
// NG: .then().catch() と async/await の混在
async function fetchData() {
    return fetch(url)
        .then(res => res.json()) // then チェーン
        .catch(err => { throw err }); // 混在
}

// OK: どちらかに統一
async function fetchData() {
    try {
        const res = await fetch(url);
        return await res.json();
    } catch (err) {
        throw new Error(`fetch failed: ${err}`);
    }
}
```

---

## カテゴリ5: モジュール解決・ESM

### HIGH

```typescript
// NG: .js 拡張子なしの相対 import を ESM ターゲットで使用
import { helper } from './utils'; // Node.js ESM で実行時エラー

// OK: .js 拡張子を明記（TypeScript でも .js を書く）
import { helper } from './utils.js';
```

```typescript
// NG: barrel file が深い再エクスポートツリーを形成（tree shaking 阻害）
// src/index.ts
export * from './components/Button';
export * from './components/Modal';
export * from './components/Form';
// ... 100 個の re-export
// → バンドラーが全てをバンドルしてしまうリスク

// OK: 必要なものだけ named export するか、直接インポートを推奨
```

### MEDIUM

```typescript
// NG: CommonJS と ESM の混在でデュアルパッケージハザード
// package.json の exports フィールドで CJS/ESM を正しく設定しないと
// CJS で require した場合と ESM で import した場合で異なるインスタンスになる
{
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "require": "./dist/index.cjs"
    }
  }
}
```

---

## カテゴリ6: Node.js 固有

### CRITICAL

```typescript
// NG: EventEmitter の listener を removeListener せずに追加（memory leak）
class Service extends EventEmitter {
    setup() {
        emitter.on('data', this.handleData); // 毎回 setup() を呼ぶとリスナーが蓄積
    }
}

// OK: cleanup を明示
class Service extends EventEmitter {
    setup() {
        this.handleData = this.handleData.bind(this);
        emitter.on('data', this.handleData);
    }
    teardown() {
        emitter.off('data', this.handleData);
    }
}
```

### HIGH

```typescript
// NG: リクエストハンドラー内で同期 fs 操作（Event Loop ブロック）
app.get('/config', (req, res) => {
    const config = fs.readFileSync('./config.json'); // ブロッキング！
    res.json(JSON.parse(config.toString()));
});

// OK: 非同期 API を使う
app.get('/config', async (req, res) => {
    const config = await fs.promises.readFile('./config.json');
    res.json(JSON.parse(config.toString()));
});
```

```typescript
// NG: Buffer.allocUnsafe() を初期化せずセキュリティ用途に使用
const buf = Buffer.allocUnsafe(32); // 前のメモリ内容が残っている可能性
const token = buf.toString('hex'); // 危険！

// OK: Buffer.alloc（ゼロ埋め）か crypto.randomBytes
import { randomBytes } from 'crypto';
const token = randomBytes(32).toString('hex');
```

```typescript
// NG: process.env.X にデフォルト値も型チェックもなし
const port = process.env.PORT; // string | undefined

// OK: 型安全な環境変数アクセス
const port = parseInt(process.env.PORT ?? '3000', 10);
// または zod/envalid で起動時に検証
```

---

## カテゴリ7: フレームワーク固有（React 検出時）

### CRITICAL

```typescript
// NG: useEffect の依存配列に関数や変数が漏れ（stale closure バグ）
function Component({ userId }: { userId: string }) {
    const [data, setData] = useState(null);

    useEffect(() => {
        fetchData(userId).then(setData); // userId が依存配列に入っていない
    }, []); // NG: userId が変わっても再 fetch されない

    // OK
    useEffect(() => {
        fetchData(userId).then(setData);
    }, [userId]); // userId を依存配列に追加
}
```

### HIGH

```typescript
// NG: useEffect がサブスクリプション/タイマーを作成して cleanup 関数を返さない
useEffect(() => {
    const timer = setInterval(tick, 1000); // コンポーネントがアンマウントされても続く
    // cleanup 関数なし → memory leak
}, []);

// OK: cleanup 関数を返す
useEffect(() => {
    const timer = setInterval(tick, 1000);
    return () => clearInterval(timer); // アンマウント時にクリーンアップ
}, []);
```

```typescript
// NG: state を直接変更（React が変更を検出しない）
const [items, setItems] = useState<Item[]>([]);
items.push(newItem); // 直接変更！
setItems(items); // 同じ参照なので React が変更を検出しない

// OK: 新しい配列を作成
setItems(prev => [...prev, newItem]);
```

```typescript
// NG: key prop に配列インデックスを使用
items.map((item, index) => (
    <Item key={index} data={item} /> // 再ソート・削除時に reconciliation が壊れる
));

// OK: 安定した一意の ID を使う
items.map(item => (
    <Item key={item.id} data={item} />
));
```

---

## カテゴリ8: フレームワーク固有（Next.js 検出時）

### HIGH

```typescript
// NG: getStaticProps + ISR で十分なのに getServerSideProps を使用
export async function getServerSideProps(context) {
    const data = await fetchBlogPosts(); // リクエストごとに fetch → パフォーマンス損失
    return { props: { data } };
}

// OK: 静的生成 + ISR を使う
export async function getStaticProps() {
    const data = await fetchBlogPosts();
    return {
        props: { data },
        revalidate: 60, // 60秒ごとに再生成
    };
}
```

```typescript
// NG: Server Component で使えるのに 'use client' を付けて JS バンドルを増やす
'use client'; // NG: このコンポーネントはインタラクティブでないのに client component になる

async function BlogPost({ id }: { id: string }) {
    const post = await fetchPost(id); // サーバーで実行できる
    return <article>{post.content}</article>;
}

// OK: 'use client' を削除し Server Component として動作させる
```

```typescript
// NG: <img> タグを直接使用（next/image の最適化を失う）
<img src="/hero.jpg" alt="Hero" />

// OK: next/image を使う
import Image from 'next/image';
<Image src="/hero.jpg" alt="Hero" width={800} height={400} />
```

### MEDIUM

```typescript
// NG: 動的ルートに generateMetadata がない（SEO・OGP の欠落）
// app/blog/[id]/page.tsx に generateMetadata なし

// OK
export async function generateMetadata({ params }: { params: { id: string } }) {
    const post = await fetchPost(params.id);
    return {
        title: post.title,
        description: post.excerpt,
        openGraph: { title: post.title, images: [post.coverImage] },
    };
}
```
