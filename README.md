# RSpec スタイルガイド

## context と describe

## FactoryGirl

- カラムの値は基本的にランダム値とする
 - 必要な値のみをテスト中で明示的に指定することにより、「このテストで重要な値はなにか」がわかりやすくなる

## 控えめなDRY

### 必要ないレコードを作らない

### update でデータを変更しない

### let を上書きしない

### shared_example, shared_context は控える

