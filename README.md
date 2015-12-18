# RSpec スタイルガイド

## context と describe

## FactoryGirl

- カラムの値は基本的にランダム値とする
 - 必要な値のみをテスト中で明示的に指定することにより、「このテストで重要な値はなにか」がわかりやすくなる

## 日付を取り扱うテストを書く

- 相対日時を利用したほうが良いケース
- 絶対日時を利用したほうが良いケース

## スコープを考慮する

- before で十分なものは let(let!) にしない
 - let(let!)で定義するものは「後で参照するもの」

## 控えめなDRY

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


- describe の直下には、その他全ての context で使用するデータのみ作成する
 - 例外は次のようなケースだが、基本的には各 context で作るほうが望ましい

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
