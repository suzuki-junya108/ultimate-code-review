# Ruby 言語固有ディープレビュー

**品質基準**: Ruby コア・Matz チーム・Rails コアチームレベル。
RuboCop + Brakeman をパスしても残る問題を検出する。

---

## カテゴリ1: Ruby イディオム

### HIGH

```ruby
# NG: frozen_string_literal: true magic comment なし
# Ruby 2.3+ で全文字列がデフォルトでミュータブル（メモリ非効率）

# OK: ファイル先頭に追加
# frozen_string_literal: true

str = "hello"
str.frozen? # => true
```

```ruby
# NG: method_missing 実装時に respond_to_missing? がない
class Proxy
  def method_missing(name, *args)
    target.send(name, *args)
  end
  # respond_to_missing? なし → respond_to? が正しく動かない
end

proxy.respond_to?(:some_method) # => false（実際には委譲できる）

# OK
class Proxy
  def method_missing(name, *args, &block)
    return super unless target.respond_to?(name)
    target.send(name, *args, &block)
  end

  def respond_to_missing?(name, include_private = false)
    target.respond_to?(name, include_private) || super
  end
end
```

```ruby
# NG: Comparable モジュールを include して <=> を未実装
class Temperature
  include Comparable
  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees
  end
  # <=> なし → >, <, between? 等が全て NoMethodError
end

# OK
class Temperature
  include Comparable
  attr_reader :degrees

  def initialize(degrees)
    @degrees = degrees
  end

  def <=>(other)
    degrees <=> other.degrees
  end
end
```

```ruby
# NG: unless に else ブランチ（非常に読みにくい）
unless condition
  do_something
else
  do_other_thing # ??? 読んでいて混乱する
end

# OK: if に書き換える
if condition
  do_other_thing
else
  do_something
end
```

### MEDIUM

```ruby
# NG: attr_accessor で不要な setter を公開
class User
  attr_accessor :name, :email, :id # id の setter は不要
end
user.id = 999 # 外部から変更できてしまう

# OK: 必要なものだけ公開
class User
  attr_reader :id
  attr_accessor :name, :email

  def initialize(id, name, email)
    @id = id
    @name = name
    @email = email
  end
end
```

---

## カテゴリ2: セキュリティ（Ruby 固有）

### CRITICAL

```ruby
# NG: eval でユーザー入力を実行
eval(params[:code]) # 任意コード実行

# OK: eval は基本的に使わない
```

```ruby
# NG: send/public_send でユーザー入力からメソッドを呼ぶ
object.send(params[:method]) # 任意メソッド呼び出し
# public_send でも危険（destroy, delete 等が呼べる）

# OK: ホワイトリストで制限
ALLOWED_METHODS = %w[name email].freeze
method_name = params[:method]
raise ForbiddenMethodError unless ALLOWED_METHODS.include?(method_name)
object.public_send(method_name)
```

```ruby
# NG: YAML.load でユーザーデータをデシリアライズ（Psych < 4.0 で RCE）
data = YAML.load(user_input) # 任意クラスのインスタンス化 → RCE

# OK: safe_load を使う
data = YAML.safe_load(user_input)
# または permitted_classes を明示（Psych 4+）
data = YAML.safe_load(user_input, permitted_classes: [Date, Symbol])
```

```ruby
# NG: Marshal.load でユーザーデータをデシリアライズ
obj = Marshal.load(user_input) # 任意コード実行

# OK: JSON 等の安全なフォーマットを使う
obj = JSON.parse(user_input)
```

```ruby
# NG: システムコマンドにユーザー入力を渡す
system("ls #{params[:path]}") # コマンドインジェクション

# OK: シェルを経由しないか、引数として渡す
system("ls", "--", params[:path]) # 引数として渡す（シェル解釈されない）
```

### HIGH

```ruby
# NG: regex で \Z を使うが \z が必要なケース（\Z は \n の前でもマッチ）
# Rails のバリデーションでよく問題になる
validates :email, format: { with: /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\Z/i }
# "user@example.com\n" もマッチしてしまう

# OK: \z を使う（文字列の絶対的な末尾）
validates :email, format: { with: /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i }
```

---

## カテゴリ3: パフォーマンス

### HIGH

```ruby
# NG: each + << で配列を構築
result = []
items.each do |item|
  result << transform(item) # each + << は O(n) で map より遅い場合がある
end

# OK: map を使う（意図も明確）
result = items.map { |item| transform(item) }
# または Symbol#to_proc
result = items.map(&:transform)
```

```ruby
# NG: Symbol#to_proc が使えるのに明示的ブロック
names = users.map { |user| user.name }
ids = items.map { |item| item.id.to_s }

# OK: Symbol#to_proc
names = users.map(&:name)
ids = items.map { |item| item.id.to_s } # to_s の引数が必要なケースは明示的ブロックでOK
```

```ruby
# NG: Array#include? を頻繁に呼ぶ（O(n)）
processed_ids = []
items.each do |item|
  next if processed_ids.include?(item.id) # O(n) の検索が毎回発生
  processed_ids << item.id
  process(item)
end

# OK: Set を使う（O(1)）
require 'set'
processed_ids = Set.new
items.each do |item|
  next if processed_ids.include?(item.id) # O(1)
  processed_ids.add(item.id)
  process(item)
end
```

---

## カテゴリ4: メモリ・GC

### HIGH

```ruby
# NG: 大ファイルを File.read で一括ロード
content = File.read('/path/to/large_file.csv') # 数GBのファイルがメモリに
lines = content.split("\n")

# OK: File.foreach でストリーミング
File.foreach('/path/to/large_file.csv') do |line|
  process(line)
end
```

```ruby
# NG: ||= をインスタンス変数のメモ化に使う（thread-safe でない）
def config
  @config ||= expensive_config_load # 複数スレッドが同時に呼ぶと複数回実行される
end

# OK: Mutex で保護
def config
  @config_mutex ||= Mutex.new
  @config_mutex.synchronize { @config ||= expensive_config_load }
end
# または ActiveSupport の thread_memoize を使う
```

---

## カテゴリ5: 並行処理・Ractor（Ruby 3+）

### HIGH

```ruby
# NG: Thread#raise の使用（安全でないスレッド割り込み）
thread = Thread.new { long_running_task }
thread.raise(RuntimeError, "Stop!") # 任意の場所で例外が発生し、クリーンアップが保証されない

# OK: フラグ変数かチャネルでキャンセルを伝える
stop_flag = Concurrent::AtomicBoolean.new(false)
thread = Thread.new do
  until stop_flag.true?
    do_work
  end
end
stop_flag.make_true
thread.join
```

```ruby
# NG: クラス変数をスレッド間で共有（race condition）
class Counter
  @@count = 0  # クラス変数はスレッド間で共有される

  def self.increment
    @@count += 1  # アトミックでない → race condition
  end
end

# OK: Thread.current に格納するか Mutex で保護
class Counter
  @mutex = Mutex.new
  @count = 0

  def self.increment
    @mutex.synchronize { @count += 1 }
  end
end
```

```ruby
# NG: Mutex で保護されていない共有ミュータブルオブジェクト
@shared_cache = {}

Thread.new { @shared_cache['key'] = compute_value } # 別スレッドが同時に読み書き
Thread.new { value = @shared_cache['key'] }

# OK: Mutex または Concurrent::Hash を使う
require 'concurrent'
@shared_cache = Concurrent::Hash.new
```

---

## カテゴリ6: テスト品質（RSpec 検出時）

### HIGH

```ruby
# NG: let を let! の代わりに使い、order-dependent なテストになっている
describe 'User#posts' do
  let(:user) { create(:user) }
  let(:post) { create(:post, user: user) } # lazy evaluation → このテストでは使われない

  it 'returns posts count' do
    expect(user.posts.count).to eq(1) # post が作られていないため 0 を返す
  end
end

# OK: let! で即座に評価（または before ブロックで明示的に作成）
describe 'User#posts' do
  let(:user) { create(:user) }
  let!(:post) { create(:post, user: user) } # 即座に作成

  it 'returns posts count' do
    expect(user.posts.count).to eq(1)
  end
end
```

```ruby
# NG: before(:all) / before(:context) で状態を設定して後処理なし
describe 'External API' do
  before(:all) do
    @api_client = ApiClient.new
    @api_client.authenticate  # 認証状態が全テストで共有される
  end
  # after(:all) なし → テスト後も認証状態が残る
end

# OK: after(:all) でクリーンアップ、またはデータベーストランザクションを使う
```

### MEDIUM

```ruby
# NG: Factory Bot で create を build_stubbed で代替できるのに DB を叩いている
describe UserPresenter do
  let(:user) { create(:user) } # DB にレコードが作成される（テストが遅くなる）
  subject { UserPresenter.new(user) }

  it 'displays name' do
    expect(subject.display_name).to eq(user.name)
  end
end

# OK: DB アクセスが不要なら build_stubbed を使う
let(:user) { build_stubbed(:user) } # DB なし、メモリ上のみ
```

---

## カテゴリ7: Rails 固有（条件付き）

### CRITICAL

```ruby
# NG: Strong Parameters (permit) なしの mass assignment（CVE 多数の根本原因）
# Rails 3 以前の attr_accessible を使わないコード
User.create(params[:user]) # password や is_admin を注入できる

# OK: strong parameters
def user_params
  params.require(:user).permit(:name, :email, :password, :password_confirmation)
end
User.create(user_params)
```

### HIGH

```ruby
# NG: before_action callback の過剰使用（暗黙の依存関係が複雑化）
class OrdersController < ApplicationController
  before_action :authenticate_user!
  before_action :load_order
  before_action :check_ownership
  before_action :check_status
  before_action :load_payment_info  # どのアクションで実行されるか追いにくい
  before_action :load_shipping_info
  # ...
end

# OK: callback を最小限にし、アクション内で明示的に呼ぶ
```

```ruby
# NG: has_and_belongs_to_many の使用（拡張性がない）
class User < ApplicationRecord
  has_and_belongs_to_many :roles  # join table にカラムを追加できない
end

# OK: has_many :through + join model を使う
class User < ApplicationRecord
  has_many :user_roles
  has_many :roles, through: :user_roles
end

class UserRole < ApplicationRecord
  belongs_to :user
  belongs_to :role
  # granted_at, granted_by 等の追加属性を持てる
end
```

```ruby
# NG: after_commit / after_save callback での外部 API 呼び出し
class Order < ApplicationRecord
  after_commit :send_notification

  private

  def send_notification
    SlackNotifier.send("Order #{id} created") # DB トランザクションと外部 API の一貫性なし
    # SlackNotifier が失敗しても order は作成済み
    # order 作成が成功しても SlackNotifier が確実に呼ばれる保証はない
  end
end

# OK: バックグラウンドジョブに委譲
after_commit :enqueue_notification_job

def enqueue_notification_job
  NotificationJob.perform_later(id)
end
```

```ruby
# NG: update_column の意図しない使用（validation・callback をスキップ）
user.update_column(:email, new_email)
# validate_email バリデーションが実行されない
# before_update / after_update callback が実行されない

# OK: 通常の update を使う（意図的にスキップする場合はコメントで説明）
user.update!(email: new_email)
# または意図的にスキップする場合
# NOTE: validation をスキップして直接更新（管理者による強制上書き）
user.update_column(:email, new_email)
```
