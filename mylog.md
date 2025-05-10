# 実装上の変更点

- **Rails バージョン変更 (`7.0.4.3` → `7.1.0`)**

  - **原因:** `concurrent-ruby` 1.3.5 以降が `Logger` を読み込まなくなったため、Rails 7.0.x で `Logger` が見つからず `NameError` が発生
  - **解決策:** Rails 7.1.0 でこの問題が修正されているため、バージョンアップで対応

- **Puma バージョン変更 (5.x 系 → `6.6`)**

  - **原因:** Rails を `7.1.0` にしたことで、プロジェクトの Rack のバージョンが Rack3 になったが、Web サーバーの Puma が 5.x 系 (Rack2 前提) のままだったため、Rack のバージョン違いによりエラー発生
  - **解決策:** Rails `7.1.0` と互換性のある Puma `6.6` に変更

- **`rails test` の挙動**

  - アクションもビューも必要
  - チュートリアルと表示が異なる (`Expected response to be a <2XX: success>, but was a <406: Not Acceptable>`)
  - ルート → コントローラにアクション追加 → ビューの順で追加していく

- **`assert_response :unprocessable_entity` でのエラー**
  - `ArgumentError: Invalid response name: unprocessable_entity` が発生
  - シンボルを HTTP ステータスコードの数値 (`422`) に変更することで解決
    - 数値指定の方が普遍的なため、シンボル名で通らない場合でも数値なら通る可能性あり？

# 感想・発見・考えたこと

## Rails の便利機能・思想

- **Scaffold:** 全部入り自動生成魔法
  - `resources` も含まれる
  - 中身:
    1.  マイグレーションファイル (テーブル作成用)
    2.  モデル
    3.  コントローラー (index, show, new, create, edit, update, destroy)
    4.  基本的なビュー (一覧・詳細・作成・編集画面)
    5.  テスト
    6.  ルーティング (`resources` 使用)
- **`rails destroy`:** `rails generate` の結果をまとめて取り消し
- **`minitest/reporters`:** テスト結果を読みやすくするための補助ヘルパー
  - RSpec にも似たようなものがあった (`.rspec` に `--format documentation`)
- **アセットパイプライン (5.1.1 章):**
  - フラットなディレクトリ構成によりファイルをより高速にブラウザに渡すことができるようになる
  - `src` 属性に `"images"` というディレクトリ名を含めないこと (以前含めたら読み込まれず困った経験あり)
- **Sass (Sassy CSS):**
  - Sassy ＝生意気な
- duck typing に確かに助けられている

## 慣習・ルール

- **パスの慣習:**
  - `root_path` -> `/` (相対パス、ビュー内のリンク用)
  - `root_url` -> `https://www.example.com/` (絶対 URL、リダイレクトやメール用)
  - 未定のリンク先 (スタブ): `"#"` を使う慣習
  - 使い分け: ビュー内のリンクには `_path` 形式、リダイレクト処理には `_url` 形式 (HTTP 標準でリダイレクト時に完全な URL が要求されるため)
- **命名規則:**
  - コントローラ名: **複数形**
  - モデル名: **単数形**

## テスト関連

- **統合テスト (Integration Test):**
  - RSpec のシステムスペックと雰囲気が似ている
  - Rails の統合テストは、より HTTP レベルに近い連携テスト
  - RSpec のシステムスペックや Rails のシステムテストは、ブラウザ操作を伴う、よりエンドユーザーに近い E2E (End-to-End) テスト

## モデル・DB 関連

- **バリデーション:**
  - `uniqueness: { case_sensitive: false }`
  - `has_secure_password` メソッドには存在性バリデーションが含まれているが、空白だけのパスワードは検知できない

## セッション・ログイン関連

- **`flash.now`:** レンダリングが終わっているページで特別にフラッシュメッセージを表示できる
- **`session` メソッドのクッキー:**
  - 原則: ブラウザを閉じると消える
  - ブラウザの「タブ復元機能」がオンだと閉じた後もセッションが残ることがある
    - これはブラウザの挙動なので Rails 側では制御不可。結果、ログアウトしないと意図せずログイン状態が続く場合あり
- **`||` と `||=` を使ったメモ化:**
  - `@変数 = @変数 || 初期化処理` も `@変数 ||= 初期化処理` も、2 回目以降の呼び出しで初期化処理（例: データベース検索）を実行しない点で、パフォーマンス向上においては同じ効果を発揮
  - **わずかな違い:**
    - `a = a || b`: `a`が真（`nil` や `false` 以外）の場合でも、`a`に`a`自身の値を再代入する処理が走る
    - `a ||= b`: `a`が真の場合は、代入処理そのものが行われない
  - `||=` の方が Ruby では一般的で簡潔
  - **`current_user` 例:** `@current_user ||= User.find_by(...)` は「`@current_user` が未設定なら DB から取得 設定済みなら何もしない」というスマートな記述

## 未分類

### Importmap による JavaScript 導入の流れ

1.  前提確認

    - `Gemfile` に `importmap-rails` があるか確認
    - レイアウト (`application.html.erb`) に `javascript_importmap_tags` があるか確認
    - `app/assets/config/manifest.js` に JS ディレクトリへのリンクがあるか確認
      - `//= link_tree ../../javascript .js`
      - `//= link_tree ../../../vendor/javascript .js`

2.  JS ファイル作成

    - `app/javascript/` 以下 (例: `app/javascript/custom/`) に JS ファイル (例: `menu.js`) を作成し、コードを記述

3.  Importmap で JavaScript を登録

    - `config/importmap.rb` に追記
      - 例: `pin_all_from "app/javascript/custom", under: "custom"` (ディレクトリごと)
      - 例: `pin "my_module", to: "my_module.js"` (ファイルごと)

4.  `application.js` からインポート

    - `app/javascript/application.js` に追記
      - 例: `import "custom/menu"` (上記 `pin_all_from` の場合)
      - 例: `import "my_module"` (上記 `pin` の場合)

# 後で詳しく調べたいことメモ

- 5.2.1 アセットパイプラインという言葉そのものについて要再確認
- パーシャルは自動生成せずに、エディタを使って手動で作成するのが一般的なのは何故
- Active Record に対応する SQL コマンド
- テスト駆動開発の red / green のリズム
- `add_index`
- `has_secure_password`
- Sass のミックスイン（mixin）機能
- `form_with` 苦手
- セッションには Session モデルがない & Active Record のモデルを使っていない
- Bootstrap の命名規則、グリッドシステム
- コントローラ用のヘルパー、モデル用の concern
