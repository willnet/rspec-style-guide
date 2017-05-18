# RSpec スタイルガイド

## describeとcontext

`describe`と`context`は同じメソッドだが、次のように使い分けることで何をテストしているのかをわかりやすくできる。

- describe の引数にはテストの対象を書く
- context の引数にはテストを実行するための条件を書く

### 例

```ruby
describe Stack do
  let!(:stack) { Stack.new }

  describe '#push' do
    context '文字列をpushしたとき' do
      before { stack.push('value') }

      it '返り値がpushした値であること' do
        expect(stack).to eq 'value'
      end
    end

    context 'nilをpushした場合' do
      it 'ArgumentErrorになること' do
        expect { stack.push(nil) }.to raise_error(ArgumentError)
      end
    end
  end

  describe '#pop' do
    context 'スタックが空の場合' do
      it '返り値はnilであること' do        
        expect(stack.pop).to be_nil
      end
    end

    context 'スタックに値があるとき' do
      before do
        stack.push 'value1'
        stack.push 'value2'        
      end

      it '最後の値を取得すること' do
        expect(stack.pop).to eq 'value2'
      end
    end
  end
end
```

## FactoryGirlのデフォルト値

FactoryGirlを利用した場合、各モデルのデフォルトのカラム値を設定することになる。このとき、各カラムの値がすべてランダム値となるように設定を書くとよい。その上で、必要な値のみをテスト中で明示的に指定することにより、「このテストで重要な値はなにか」がわかりやすくなる。

### よくない例

アカウントが有効化されているかどうかをactiveカラムで管理しているとする。このような、有効／無効を表すカラムが固定されているケースはよく見かける。

```ruby
FactoryGirl.define do
  factory :user do
    name 'willnet'
    active true
  end
end
```

```ruby
describe User, type: :model do
  describe '#send_message' do
    let!(:sender) { create :user, name: 'maeshima' }
    let!(:receiver) { create :user, name: 'kamiya' }

    it 'メッセージが正しく送られること' do
      expect { sender.send_message(receiver: receiver, body: 'hello!') }
        .to change { Message.count }.by(1)
    end
  end
end
```

このテストは`User#active`が`true`であることが暗黙的な条件になってしまっている。`sender.active #=> false`のときや`receiver.active #=> false`のときにどう振る舞うかを伝えることができていない。

さらに、このテストでは`name`を明示的に指定しているがこれは必要な指定なのだろうか？テストを読む人に余計なリソースを消費させてしまう、無駄なデータ指定はなるべく避けるのが好ましい。

### よい例

```ruby
FactoryGirl.define do
  factory :user do
    sequence(:name) { |i| "test#{i}"}
    active { [true, false].sample }
  end
end
```

```ruby
describe User, type: :model do
  describe '#send_message' do
    let!(:sender) { create :user }
    let!(:receiver) { create :user }

    it 'メッセージが正しく送られること' do
      expect { sender.send_message(receiver: receiver, body: 'hello!') }
        .to change { Message.count }.by(1)
    end
  end
end
```

このテストだと「`User#active`の戻り値が`User#send_message`の動作に影響しない」ということが(暗黙的にであるが)伝わる。もし`User#active`が影響するような修正が加えられた場合、CIで時々テストが失敗することによって、テストが壊れたことに気付けるはずだ。

## FactoryGirlで一対多以上の関連をデフォルトで作成しない

FactoryGirlでモデルを作成する際に、関連しているモデルも同時に作成することができる。

belongs_toやhas_oneでの関連であれば特に問題はないが、has_manyの関連を扱う場合には注意が必要になる。

例として、UserとPostが一対多としてUserのFactoryGirlでの定義を書いてみる。

```ruby
FactoryGirl.define do
  factory :user do
    sequence(:name) { |i| "username#{i}" }

    after(:create) do |user, evaluator|
      create_list(:post, 2, user: user)
    end
  end
end
```

`after(:create)`を使い、Userが作成された際に関連するPostも作成されるようにした。この定義を用いてUserが投稿したPostのうち、人気順で返す`User#posts_ordered_by_popularity`のテストを書いてみる。


```ruby
RSpec.describe User, type: :model do
  describe '#posts_ordered_by_popularity' do
    let!(:user) { create(:user) }
    let!(:post_popular) do
      post = user.posts[0]
      post.update(popularity: 5)
      post
    end
    let!(:post_not_popular) do
      post = user.posts[1]
      post.update(popularity: 1)
      post
    end

    it 'return posts ordered by populality' do
      expect(user.posts_ordered_by_popularity).to eq [post_popular, post_not_popular]
    end
  end
end
```

理解しづらいテストコードになった。このテストはUserのレコードを作成した時に、関連するPostを2つ作成することに依存している。また、[update](https://github.com/willnet/rspec-style-guide#updateでデータを変更しない)を利用してデータを変更しているため、最終的なレコードの状態を把握しづらくなっている。

これを避けるには、まず、デフォルトで一対多の関連レコードを作成することをやめるとよい。

```ruby
FactoryGirl.define do
  factory :user do
    sequence(:name) { |i| "username#{i}" }

    trait(:with_posts) do
      after(:create) do |user, evaluator|
        create_list(:post, 2, user: user)
      end
    end
  end
end
```

`trait`を利用し、デフォルトではPostを作成しないようにした。どんな値でも良いので関連先のPostがほしいときにはtraitを指定しUserを作成すれば良い。

関連先はテスト中で明示的に作成するようにする。

```ruby
RSpec.describe User, type: :model do
  describe '#posts_ordered_by_popularity' do
    let!(:user) { create(:user) }
    let!(:post_popular) { create :post, user: user, popularity: 5 }
    let!(:post_not_popular) { create :post, user: user, popularity: 1 }

    it 'return posts ordered by populality' do
      expect(user.posts_ordered_by_popularity).to eq [post_popular, post_not_popular]
    end
  end
end
```

これで、最初の例よりだいぶ見やすくなった。

## 日時を取り扱うテストを書く

日時を取り扱うテストを書く場合、絶対時間を使う必要がないケースであればなるべく現在日時からの相対時間を利用するのが良い。その方が実装の不具合に気づける可能性が増すからだ。

例として、先月に公開された投稿を取得するscopeと、そのテストを絶対時間を用いて記述する。

```ruby
class Post < ApplicationRecord
  scope :last_month_published, -> { where(publish_at: (Time.zone.now - 31.days).all_month) }
end
```

```ruby
require 'rails_helper'

RSpec.describe Post, type: :model do
  describe '.last_month_published' do
    let!(:april_1st) { create :post, publish_at: Time.zone.local(2017, 4, 1) }
    let!(:april_30th) { create :post, publish_at: Time.zone.local(2017, 4, 30) }

    before do
      create :post, publish_at: Time.zone.local(2017, 5, 1)
      create :post, publish_at: Time.zone.local(2017, 3, 31)
    end

    it 'return published posts in last month' do
      Timecop.travel(2017, 5, 6) do
        expect(Post.last_month_published).to contain_exactly(april_1st, april_30th)
      end
    end
  end
end
```

このテストは常に成功するが、実装にはバグが含まれている。

テストを相対日時に変更してみる。

```ruby
require 'rails_helper'

RSpec.describe Post, type: :model do
  describe '.last_month_published' do
    let!(:now) { Time.zone.now }
    let!(:last_beginning_of_month) { create :post, publish_at: 1.month.ago(now).beginning_of_month }
    let!(:last_end_of_month) { create :post, publish_at: 1.month.ago(now).end_of_month  }

    before do
      create :post, publish_at: now
      create :post, publish_at: 2.months.ago(now)
    end

    it 'return published posts in last month' do
      expect(Post.last_month_published).to contain_exactly(last_beginning_of_month, last_end_of_month)
    end
  end
end
```

このテストは、例えば3月1日に実行すると失敗する。常にバグを検知できるわけではないが、CIを利用することで、ずっと不具合が残り続ける可能性を減らすことができるだろう。


## 日付を外部から注入する

[「現在時刻」を外部入力とする設計と、その実装のこと - クックパッド開発者ブログ](http://techlife.cookpad.com/entry/2016/05/30/183947)

## beforeとlet(let!)の使い分け

テストの前提となるオブジェクト(やレコード)を生成する場合、let(let!)やbeforeを使う。このとき、生成後に参照するものをlet(let!)で生成し、それ以外をbeforeで生成すると可読性が増す。

次のような`scope`を持つ`User`モデルがあるとする。

```ruby
class User < ApplicationRecord
  scope :active, -> { where(deleted: false).where.not(confirmed_at: nil) }
end
```

このテストを`let!`のみを用いて書くと次のようになる。

```ruby
require 'rails_helper'

RSpec.describe User, type: :model do
  describe '.active' do
    let!(:active) { create :user, deleted: false, confirmed_at: Time.zone.now }
    let!(:deleted_but_confirmed) { create :user, deleted: true, confirmed_at: Time.zone.now }
    let!(:deleted_and_not_convirmed) { create :user, deleted: true, confirmed_at: nil }
    let!(:not_deleted_but_not_confirmed) { create :user, deleted: false, confirmed_at: nil }

    it 'return active users' do
      expect(User.active).to eq [active]
    end
  end
end
```

`let!`と`before`を併用して書くと次のようになる。

```ruby
require 'rails_helper'

RSpec.describe User, type: :model do
  describe '.active' do
    let!(:active) { create :user, deleted: false, confirmed_at: Time.zone.now }

    before do
      create :user, deleted: true, confirmed_at: Time.zone.now
      create :user, deleted: true, confirmed_at: nil
      create :user, deleted: false, confirmed_at: nil
    end

    it 'return active users' do
      expect(User.active).to eq [active]
    end
  end
end
```

後者のほうが「メソッドの戻り値となるオブジェクト」と「それ以外」を区別しやすく、見やすいコードとなる。

※`let!(:deleted_but_confirmed)`のように名前をつけることで、どんなレコードなのか理解しやすくなると感じる人もいるかもしれない。しかしレコードに名前付けが必要であれば、単純にコメントとして補足してやればよいだろう


## 控えめなDRY

DRYにする行為は常に善だと思われているかもしれないが、そうではない。例えばコードの重複を一つにまとめるということは処理の抽象化をするということで、状況や抽象化の仕方によってはDRYによって減らされたコストを上回るコストが発生することもある。

### shared_examplesはよく考えて使う

shared_examplesを利用するとコードの重複を削除できるが、書き方によってはかえって可読性を落とすことがある。


例として、引数として渡された曜日に対応した分だけポイントを増やすメソッド`Point#increase_by_day_of_the_week`のテストを`shared_examples`を利用して書いてみる。shard_exampleの定義は別ファイルに書かれているとして、先にshared_examplesを利用する側だけのコードを見てみる。

```ruby
RSpec.describe Point, type: :model do
  describe '#increase_by_day_of_the_week' do
    let(:point) { create :point, point: 0 }

    it_behaves_like 'point increasing by day of the week', 100 do
      let(:wday) { 0 }
    end

    it_behaves_like 'point increasing by day of the week', 50 do
      let(:wday) { 1 }
    end

    it_behaves_like 'point increasing by day of the week', 30 do
      let(:wday) { 2 }
    end

    # ...
  end
end
```

どんな前提条件で結果として何を期待しているのか、これだけを見て理解できるだろうか。

定義は次のようになる。

```ruby
RSpec.shared_examples 'point increasing by day of the week' do |expected_point|
  it "increase by #{expected_point}" do
    expect(point.point).to eq 0
    point.increase_by_day_of_the_week(wday)
    expect(point.point).to eq expected_point
  end
end
```

このテストが読みづらいのは、shared_examplesを成立させるために必要な前提条件が多いことが一つ挙げられる。

- テストされる主体である`point`
- メソッドに渡される引数である`wday`
- 期待する結果である`expected_point`

また、それぞれの定義が分散していることが挙げられる

- 外側のlet(point)
- `it_behaves_like`ブロック内のlet(wday)
- `it_behaves_like`の第二引数(expected_point)

可読性を上げるには、まず、適切に名前をつけることが考えられるだろう。

```ruby
RSpec.shared_examples 'point increasing by day of the week' do |expected_point:|
  it "increase by #{expected_point}" do
    expect(point.point).to eq 0
    point.increase_by_day_of_the_week(wday)
    expect(point.point).to eq expected_point
  end
end

RSpec.describe Point, type: :model do
  describe '#increase_by_day_of_the_week' do
    let(:point) { create :point, point: 0 }

    context 'on sunday' do
      let(:wday) { 0 }
      it_behaves_like 'point increasing by day of the week', expected_point: 100
    end

    context 'on monday' do
      let(:wday) { 1 }
      it_behaves_like 'point increasing by day of the week', expected_point: 50
    end

    context 'on tuesday' do
      let(:wday) { 2 }
      it_behaves_like 'point increasing by day of the week', expected_point: 30
    end

    # ...
  end
end
```

新しくcontextを作り、`wday`についての説明を加えた。そして、期待する結果である`expected_point`をキーワード引数として利用することで、実引数側に名前がつき、数値が何を表すのかすぐに理解できるようになった。

しかし、そもそもこれは`shared_examples`を利用すべきケースなのだろうか？`shared_examples`を利用せずに書くと次のようになる。

```ruby
RSpec.describe Point, type: :model do
  describe '#increase_by_day_of_the_week' do
    let(:point) { create :point, point: 0 }

    context 'on sunday' do
      let(:wday) { 0 }

      it "increase by 100" do
        expect(point.point).to eq 0
        point.increase_by_day_of_the_week(wday)
        expect(point.point).to eq 100
      end
    end

    context 'on monday' do
      let(:wday) { 1 }

      it "increase by 50" do
        expect(point.point).to eq 0
        point.increase_by_day_of_the_week(wday)
        expect(point.point).to eq 50
      end
    end

    context 'on tuesday' do
      let(:wday) { 2 }

      it "increase by 30" do
        expect(point.point).to eq 0
        point.increase_by_day_of_the_week(wday)
        expect(point.point).to eq 30
      end
    end

    # ...
  end
end
```

`it_behaves_like`での前提条件や引数が多くなればなるほど複雑度が増す。DRYにするメリットが複雑度を上回るかどうか、慎重に考えるべきだ。

## スコープを考慮する
### describe 外にテストデータを置かない

例えば、次のような spec があるとする

```ruby
describe 'sample specs' do
  context 'a' do
    # ...
  end

  context 'b' do
    let!(:need_in_b_and_c) { ... }        
    # ...
  end

  context 'c' do
    let!(:need_in_b_and_c) { ... }        
    # ...
  end
end
```

この場合、b と c で同じ前提条件を利用しているので、一つ上のレベルに移動してDRYにしようと考える人もいるかもしれない。

```ruby
describe 'sample specs' do
  let!(:need_in_b_and_c) { ... }        

  context 'a' do
    # ...
  end

  context 'b' do
    # ...
  end

  context 'c' do
    # ...
  end
end
```

しかし、これは良くない。'a' のコンテキストに、必要のない前提条件が含まれてしまうからだ。この例だけだとピンとこないかもしれない。このような let! が 10、context が 30 あって、どの let! がどの context に対応する前提条件なのかわからない状況を想像すると、下手に前提条件をまとめる怖さがわかるだろうか。

もちろん、すべての context において、共通で使う前提条件であれば、まとめてしまうのは問題ない。


### 各ブロックにおける前提条件の配置ルール

- 各ブロックの前提条件は、その配下のすべての expectation で利用するもののみ書く
- 特定の expectation のみで利用するものにおいては、その expectation に書く


### 落穂拾い

- 各ブロックにおける前提条件の配置ルール、の例外は次のようなケース
- だが、基本的には各ブロック内で宣言するほうが望ましい

```ruby
let!(:user) { create :user, enabled: enabled }

context 'when user is enabled' do
  let(:enabled) { true }
  it { ... }
end

context 'when user is disabled' do
  let(:enabled) { false }
  it { ... }
end

```


### 必要ないレコードを作らない

パフォーマンスの観点から、レコードを作らなくてすむ場合は作らないようにしたい。

```ruby
describe 'posts#index' do
  context 'when visit /posts' do
    let!(:posts) { create_list :post, 100 }

    before { visit posts_path }

    it 'display all post titles' do
      posts.each do |post|
        expect(page).to have_content post.title
      end
    end
  end
end
```

「100件の投稿タイトルが表示できること」をテストしたい場合は別だが、ただ投稿タイトルを表示できているかチェックできればいい場合、明らかに無駄なレコードを作っている。

この場合の最小限のレコード数は1件である。

```ruby
describe 'posts#index' do
  context 'when visit /posts' do
    let!(:post) { create :post }

    before { visit posts_path }

    it 'display post title' do
      expect(page).to have_content post.title
    end
  end
end
```

モデルのユニットテストでも、作らなくてよいレコードを作っているケースはよくある。

```ruby
describe 'User' do
  describe '#fullname' do
    let!(:user) { create :user, first_name: 'Shinichi', last_name: 'Maeshima' }

    it 'return full name' do
      expect(user.fullname).to eq 'Shinichi Maeshima'
    end
  end  
end
```

`User#fullname`はレコードが保存されているか否かに影響しないメソッドである。この場合は`create`ではなく`build`(もしくは`build_stubbed`)を使う。

```ruby
describe 'User' do
  describe '#fullname' do
    let!(:user) { build :user, first_name: 'Shinichi', last_name: 'Maeshima' }

    it 'return full name' do
      expect(user.fullname).to eq 'Shinichi Maeshima'
    end
  end  
end
```

このような単純なケースでは`User.new`を利用しても良い。

### updateでデータを変更しない

FactoryGirlで作成したレコード中のカラムをupdateメソッドで変更すると、最終的なレコードの状態がわかりにくくなるし、テストに依存している属性もわかりにくくなるので避ける。

```ruby
describe Post do
  let!(:post) { create :post }

  describe '#published?' do
    subject { post.published? }

    context 'when the post has already published' do
      it { is_expected.to eq true }
    end

    context 'when the post has not published' do
      before { post.update(publish_at: nil) }

      it { is_expected.to eq false }
    end    

    context 'when the post is closed' do
      before { post.update(status: :close) }

      it { is_expected.to eq false }
    end    

    context 'when the title includes "[WIP]"' do
      before { post.update(title: '[WIP]hello world') }

      it { is_expected.to eq false }
    end    
  end    
end
```

`Post#published?`メソッドに依存している属性をすぐに理解することができるだろうか？updateはたいていFactoryGirlのデフォルト値を「一番データとして多い形」に設定し、それを少し変更して使うために使われる。

updateは使用せず、[FactoryGirlのデフォルト値)](https://github.com/willnet/rspec-style-guide#factorygirlのデフォルト値)に記載したようにデフォルト値をランダムに保つと良い。

### letを上書きしない

`let`で定義したパラメータを内側のcontextで上書きすると、[updateでデータを変更しない](https://github.com/willnet/rspec-style-guide#updateでデータを変更しない)で説明した例と同様に、最終的なレコードの状態がわかりにくくなるので避ける。

```ruby
describe Post do
  let!(:post) { create :post, title: title, status: status, publish_at: publish_at }
  let(:title) { 'hello world' }
  let(:status) { :open }
  let(:publish_at) { Time.zone.now }

  describe '#published?' do
    subject { post.published? }

    context 'when the post has already published' do
      it { is_expected.to eq true }
    end

    context 'when the post has not published' do
      let(:publish_at) { nil }

      it { is_expected.to eq false }
    end    

    context 'when the post is closed' do
      let(:status) { :close }

      it { is_expected.to eq false }
    end    

    context 'when the title includes "[WIP]"' do
      let(:title) { '[WIP]hello world'}

      it { is_expected.to eq false }
    end    
  end    
end
```

### subjectを使うときの注意事項

`subject`は`is_expected`や`should`を使い一行でexpectationを書く場合は便利だが、逆に可読性を損なう使われ方をされる場合がある。

```ruby
describe 'ApiClient#save_record_from_api' do
  let!(:client) { ApiClient.new }
  subject { client.save_record_from_api(params) }

  #
  # ...多くのexpectationを省略している...
  #

  context 'when pass  { limit: 10 }' do
    let(:params) { { limit: 10} }

    it 'return ApiResponse object' do
      is_expected.to be_an_instance_of ApiResponse
    end

    it 'save 10 items' do
      expect { subject }.to change { Item.count }.by(10)
    end
  end
end
```

このようなとき、`expect { subject }`の`subject`は一体何を実行しているのかすぐには判断できず、ファイルのはるか上方にある`subject`の定義を確認しなければならない。

そもそも"subject"は名詞であり、副作用を期待する箇所で定義すると混乱を招く。

`is_expected`を利用し暗黙の`subject`を利用する箇所と直接`subject`を明示する箇所が混在しており、どうしても`subject`を使いたい場合は、`subject`に名前をつけて使うと良い。

```ruby
describe 'ApiClient#save_record_from_api' do
  let!(:client) { ApiClient.new }
  subject(:execute_api_with_params) { client.save_record_from_api(params) }

  context 'when pass  { limit: 10 }' do
    let(:params) { { limit: 10} }

    it 'return ApiResponse object' do
      is_expected.to be_an_instance_of ApiResponse
    end

    it 'save 10 items' do
      expect { execute_api_with_params }.to change { Item.count }.by(10)
    end
  end
end
```

`expect { subject }`の時よりはわかりやすくなったはずだ。

`is_expected`を利用していない場合は、`subject`の利用をやめて`client.save_record_from_api(params)`を各expectationにべた書きするのが良い。

### allow_any_instance_ofを避ける

[公式のドキュメント](https://relishapp.com/rspec/rspec-mocks/docs/working-with-legacy-code/any-instance)にも書かれているが、`allow_any_instance_of`(`expect_any_instance_of`)が必要な時点でテスト対象の設計がおかしい可能性がある。

例として、次のような`Statement#issue`のテストを書いてみる。

```ruby
class Statement
  def issue(body)
    client = TwitterClient.new
    client.issue(body)
  end
end
```

```ruby
describe Statement do
  describe '#issue' do
    let!(:statement) { Statement.new }

    it 'call TwitterClient#issue' do
      expect_any_instance_of(TwitterClient).to receive(:issue).with('hello')
      statement.issue('hello')
    end
  end
end
```

`expect_any_instance_of`を使ってしまったのは、`Statement`クラスと`TwitterClient`クラスが密結合しているのが原因である。結合を弱めてみる。

```ruby
class Statement
  def initialize(client: TwitterClient.new)
    @client = client
  end

  def issue(body)
    client.issue(body)
  end

  private

  def client
    @client
  end
end
```

```ruby
describe Statement do
  describe '#issue' do
    let!(:client) { double('client') }
    let!(:statement) { Statement.new }

    it 'call TwitterClient#issue' do
      expect(client).to receive(:issue).with('hello')
      statement.issue('hello')
    end
  end
end
```

`issue`メソッドを持つオブジェクトであれば、どのクラスでも`client`として扱えるように修正した。外部から`client`を指定できる作りにしたことで、将来的に`FacebookClient`など別のクライアントにも対応できるようになった。結合が弱まり、単純なモックオブジェクトでテストが記述できるようになった。
