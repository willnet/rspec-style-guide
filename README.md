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

- describe 外にテストデータを置かない
- describe の直下には、その他全ての context で使用するデータのみ作成する
 - 例外は次のようなケースだが、基本的には各 context で作るほうが望ましい

```
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

