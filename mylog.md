- Rails7.0.4.3 では`logger`読み込み問題が起きたので 7.1.0 を採用

  - `concurrent-ruby`1.3.5〜が`Logger`を読み込まなくなったため、
    Rails 7.0.x が`Logger`を見つけられず`NameError`が発生
  - Rails 7.1.0 ではこの問題が修正されているとのことで、バージョンアップで解決

- puma のエラーが起きたので Rails7.1.0 と互換性のある puma6.6 を採用

  - Rails を 7.1.0 にしたことで、プロジェクトの Rack のバージョンが Rack3 になった
  - しかし Web サーバーの Puma が 5.x 系(Rack2 前提)のままだった
  - Rack のバージョンが違うことによりエラー発生
  - Rails7.1.0 と互換性のある puma6.6 に変えることで解決

- scaffold、全部入り自動生成魔法

  - `resources` も含まれてる！
  - 中身
    1. マイグレーションファイル(テーブル作成用)
    2. モデル
    3. コントローラー(index, show, new, create, edit, update, destroy)
    4. 基本的なビュー(一覧・詳細・作成・編集画面)
    5. テスト
    6. ルーティング(`resources`使用)

- `rails destroy`で`rails generate`をまとめて取り消し

- `rails test`には、アクションもビューも必要

  - チュートリアルと表示が異なる
  - `Expected response to be a <2XX: success>, but was a <406: Not Acceptable>`

- `minitest/reporters`は読みやすくするための補助ヘルパー
  - RSpec にも似たようなものがあった
