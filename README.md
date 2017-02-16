# RSpec スタイルガイド

## context と describe

describe と context は同じメソッドだが、次のように使い分けることで何をテストしているのかをわかりやすくできる。

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

## FactoryGirl

- カラムの値は基本的にランダム値とする
 - 必要な値のみをテスト中で明示的に指定することにより、「このテストで重要な値はなにか」がわかりやすくなる

### よくない例

```ruby
FactoryGirl.define do
  factory :user do
    sequence(:name) { |i| "test#{i}"}
    active true
  end
end
```

```ruby
describe User, type: :model do
  describe '#make_comment' do # active が false だとコケるメソッドの方が良いのでは
    let!(:user) { create :user }

    it 'コメントが作成されること' do
      expect { user.make_comment }.to change { Comment.count }.by(1)
    end
  end
end
```

これは active が true であることが暗黙的な条件になってしまっている。

メモ: もっとデフォルト値に依存しているけど依存していることがわかりづらい例にしたい

### よい例

## 日付を取り扱うテストを書く

- 相対日時を利用したほうが良いケース
- 絶対日時を利用したほうが良いケース

## 日付を外部から注入する

## スコープを考慮する

- before で十分なものは let(let!) にしない
 - let(let!)で定義するものは「後で参照するもの」

## 控えめなDRY

- テストはDRYにしない方が良い
- コードが重複していればDRYにして良いかというと、そうではないケースもある
  - 実例ほしい！
- 意味的に重複していればOK

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

### update でデータを変更しない

### let を上書きしない

### shared_example, shared_context は控える

### subject を使うときの注意事項

### allow_any_instance_of を避ける
