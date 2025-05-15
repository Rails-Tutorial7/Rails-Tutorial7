# Rails チュートリアル学習記録 一周目投げっぱなし Ver.

## I. 実装上の変更点・トラブルシューティング

### バージョン関連

- **Rails バージョン変更 (`7.0.4.3` → `7.1.0`)**

  - **原因:** `concurrent-ruby` 1.3.5 以降が `Logger` を読み込まなくなったため、Rails 7.0.x で `Logger` が見つからず `NameError` が発生
  - **解決策:** Rails 7.1.0 でこの問題が修正されているため、バージョンアップで対応

- **Puma バージョン変更 (5.x 系 → `6.6`)**
  - **原因:** Rails を `7.1.0` にしたことで、プロジェクトの Rack のバージョンが Rack3 になったが、Web サーバーの Puma が 5.x 系 (Rack2 前提) のままだったため、Rack のバージョン違いによりエラー発生
  - **解決策:** Rails `7.1.0` と互換性のある Puma `6.6` に変更

### テスト関連のエラーと対処

- **`rails test` の挙動について**

  - アクションもビューも必要
  - チュートリアルと表示が異なる (`Expected response to be a <2XX: success>, but was a <406: Not Acceptable>`)
  - ルート → コントローラにアクション追加 → ビューの順で追加していく

- **`assert_response :unprocessable_entity` でのエラー**

  - `ArgumentError: Invalid response name: unprocessable Entity` が発生
  - シンボルを HTTP ステータスコードの数値 (`422`) に変更することで解決
    - 数値指定の方が普遍的なため、シンボル名で通らない場合でも数値なら通る可能性あり？

- **`ActiveRecord::StatementInvalid: SQLite3::BusyException: database is locked` エラー**
  - `test/test_helper.rb` 内の `parallelize(workers: :number_of_processors, with: :threads)` をコメントアウトすることで解決 (並列実行に起因する SQLite のロック問題)

## II. Rails の便利機能・思想・用語

### Rails の基本機能

- **Scaffold:**
  - 「足場」の意味 開発の雛形を自動生成する強力なコマンド
  - 生成されるもの:
    1.  マイグレーションファイル (テーブル定義)
    2.  モデル
    3.  コントローラー (CRUD アクション一式: `index`, `show`, `new`, `create`, `edit`, `update`, `destroy`)
    4.  基本的なビュー (一覧、詳細、作成、編集画面)
    5.  各種テストファイル
    6.  ルーティング (`resources` を使用)
- **`rails destroy`:**
  - `rails generate` で生成したファイル群をまとめて削除（取り消し）できる
- **アセットパイプライン (Asset Pipeline):** (5.1.1 章)
  - 本番環境で CSS や JavaScript ファイルを結合・圧縮し、ブラウザへの配信を効率化する仕組み
  - 開発時は個別のファイルとして扱えるが、本番では最適化される
  - ビューで画像を参照する際、`image_tag` ヘルパーを使えば、アセットパイプラインがよしなにしてくれるので、`src` 属性に `"images/"` のようなディレクトリ名を含める必要はない (含めると読み込まれないことがある)
- **Sass (Sassy CSS):**
  - CSS をより効率的に書くための拡張言語 「Sassy」は「生意気な」という意味合い 変数やネスト、ミックスインなどの機能が使える
- **`rails db:seed`:**
  - `db/seeds.rb` ファイルに記述された初期データ（サンプルデータ）をデータベースに投入するコマンド 開発初期やテストデータの準備に便利

### Ruby の概念・記法 (Rails でよく使われるもの)

- **ダックタイピング (Duck Typing):**
  - オブジェクトの実際の型よりも、そのオブジェクトが特定のメソッドを持っているかどうか（振る舞えるか）を重視
  - Rails ではこの思想により柔軟なコードが書ける場面がある
- **メモ化 (`||=` を使ったテクニック):**
  - `@変数 ||= 初期化処理` の形で、インスタンス変数が `nil` または `false` の場合にのみ初期化処理を実行し、その結果を変数に代入する
  - 2 回目以降の呼び出しでは既に値がセットされているため、初期化処理（例: 負荷の高いデータベース検索）をスキップでき、パフォーマンス向上に繋がる
  - `@変数 = @変数 || 初期化処理` とのわずかな違い:
    - `a = a || b`: `a`が真（`nil` や `false` 以外）の場合でも、`a`に`a`自身の値を再代入する処理が内部的に発生する
    - `a ||= b`: `a`が真の場合は、代入処理そのものが実行されない
  - `current_user` の実装例 (`@current_user ||= User.find_by(...)`) は、「`@current_user` がまだ設定されていなければデータベースからユーザー情報を取得し、既に設定済みならその値をそのまま使う」という効率的な処理の典型例
- **クラスメソッドとインスタンスメソッド:**
  - **クラスメソッド:**
    - 定義: メソッド名の前にクラス名 (`ClassName.method_name`) または `self.` (クラス定義の直下で使う場合 `self.method_name`) を付ける
    - 呼び出し: クラス名をレシーバーとして呼び出す (`ClassName.method_name`)
    - 用途: クラス全体に関連する処理、汎用的なユーティリティ、特定のインスタンスの状態に依存しない処理
  - **インスタンスメソッド:**
    - 定義: メソッド名の前に何も付けない (`method_name`)
    - 呼び出し: クラスのインスタンス（オブジェクト）をレシーバーとして呼び出す (`instance.method_name`)
    - 用途: 特定のインスタンスの状態を操作したり、インスタンスの属性に依存した処理
- **`super` キーワード:**
  - クラスの継承関係において、子クラスのメソッド内から、親クラスにある同じ名前のメソッドを呼び出すための Ruby のキーワード
  - メソッドのオーバーライド（上書き）と密接に関連 子クラスで親クラスの同名メソッドを定義した際に、親の処理を呼び出しつつ独自の処理を追加したい場合などに使う
  - RSpec の`before do`や Minitest の`setup`のようなテストフレームワークのフック機能とは異なり、`super`は Ruby 言語自体の機能
- **早期脱出 (Early Exit / Guard Clause):**
  - メソッドの冒頭で、特定の条件を満たさない場合にすぐに処理を終了させる (`return` する) ことで、ネストが深い `if` 文を避け、コードを読みやすくするテクニック
- **三項演算子:**
  - `条件式 ? 真の場合の値 : 偽の場合の値` の形式で、簡単な `if-else` を一行で簡潔に書ける
- **シンボルと文字列:**
  - Rails ではハッシュのキーなどにシンボルがよく使われる パフォーマンスが良い、イミュータブルであるなどの特徴がある
- Rails はデフォルトでは外部キーの名前を`<class>_id`として理解し、`<class>`に当たる部分からクラス名（正確には小文字に変換されたクラス名）を推測

### Rails の慣習・ルール

- **命名規則:**
  - コントローラ名: **複数形** (例: `UsersController`)
  - モデル名: **単数形** (例: `User`)
- **パスの使い分け:**
  - `_path` 形式 (例: `root_path`): 相対パスを生成 主にビュー内のリンク (`link_to` など) で使用 結果は `/` や `/users` のようになる
  - `_url` 形式 (例: `root_url`): 絶対 URL を生成 主にリダイレクト処理 (`redirect_to`) やメール本文内のリンクで使用 HTTP プロトコルの標準でリダイレクト時には完全な URL が要求されるため 結果は `http://localhost:3000/` のようになる
  - 未定のリンク先 (スタブ): `"#"` を使うのが一般的な慣習
- **if 文のスタイル:**
  - 処理が 1 行の場合は後置 `if` (例: `action if condition`)
  - 処理が複数行に渡る場合は前置 `if` (例: `if condition ... end`)

### モデル・データベース関連

- **バリデーション:**
  - `uniqueness: { case_sensitive: false }`: 大文字・小文字を区別せずに一意性を検証
  - `has_secure_password` メソッド:
    - これ自体に存在性バリデーションが含まれている
    - ただし、空白文字のみのパスワードは検知できない (別途バリデーションが必要な場合あり)
    - `has_secure_password` による存在性バリデーションがあるため、`validates :password, presence: true, allow_nil: true` のように `allow_nil: true` を設定しても、新規ユーザー登録時にパスワードが空のまま通ることはない (ユーザー更新時など、パスワード変更が任意の場合に `allow_nil` が役立つ)
- **`save` vs `update_attribute` (古い) vs `update_attributes` (古い) vs `update`:**
  - `save`: オブジェクトの現在の状態をデータベースに保存 新規作成時は INSERT、既存オブジェクトの場合は変更のあった属性のみを UPDATE する バリデーションを実行する
  - `update_attribute(:name, "new_name")`: (非推奨に近い) 特定の属性 1 つだけを更新し、即座に DB に保存 バリデーションをスキップする
  - `update_attributes(name: "new_name", email: "new@example.com")`: (非推奨) 複数の属性を一度に更新し、即座に DB に保存 バリデーションを実行する Rails 4 以降は `update` が推奨
  - `update(name: "new_name", email: "new@example.com")`: `update_attributes` と同様の機能を持つ、より推奨される書き方 バリデーションを実行する
- **`default_scope`:**
  - モデルからデータを取得する際のデフォルトの並び順などを指定する
  - 例: `default_scope -> { order(created_at: :desc) }` (作成日時の降順をデフォルトにする)
  - `-> { ... }` はラムダ (無名関数) の記法
- **`new` と `build` の違い (関連付けにおける):**

  - `Micropost.new(...)`: `Micropost` クラスの新しいインスタンスをメモリ上に作成 この時点では `user_id` は `nil`
  - `@user.microposts.build(...)`: `@user` に紐付いた `Micropost` クラスの新しいインスタンスをメモリ上に作成 この時点で、新しいマイクロポストの `user_id` には `@user.id` が自動的に設定される
  - どちらもデータベースにはまだ保存されず、保存するには `.save` メソッドが必要

- `index` = 検索を高速化する秘密兵器

  - **インデックスなし**: 全データチェック -> 遅い
  - **インデックスあり**: 特定カラムの索引（B-tree）を利用 -> 速い
    - 索引を見て、データ本体の場所へジャンプ
  - B-tree: 効率的な検索のためのデータ構造
    - 値がソートされて格納
    - 階層をたどり、少ない比較回数で目的のデータに到達
    - データ量が増えても検索速度の低下は緩やか

- `has_many :through`

  - `user.following`だけで、フォローしているユーザーのリストを簡単に取得できる
    - **多対多**: ユーザーは多くの人をフォローでき、多くの人にフォローされる
    - **中間テーブル**: `relationships`テーブル（`follower_id`, `followed_id`を持つ）が、誰と誰が繋がっているかの「関係性」を記録
    - **`has_many :through`**: この中間テーブルを経由して、フォローしている/されている「ユーザー」の情報を取得する仕組み
    - `user.following` の裏側 (Rails の動き)
      - `user.following` を呼び出すと、Rails は 2 段階で動くイメージ
        1.  **関係性の取得**:
        - まず`user.active_relationships`(`has_many :active_relationships, class_name: "Relationship", foreign_key: "follower_id"`で定義された関連)を実行
        - SQL イメージ: `SELECT * FROM relationships WHERE relationships.follower_id = #{user.id}`
        - これで、そのユーザーが「誰を」フォローしているかの「関係性」レコード（`Relationship`オブジェクトの集まり）を relationships テーブルから取得
        2.  **ユーザー情報の取得**:
        - 1 で取得した各「関係性」レコードの `followed_id` を使う
        - SQL イメージ: `SELECT * FROM users WHERE users.id IN (#{followed_idsのリスト})`
        - これで実際にフォローされているユーザーの情報を users テーブルから取得

- Rails では`includes`メソッドで eager loading を実現
  - N+1 問題への対処

### コントローラ・ルーティング関連

- **`resources` ルーティング:**
  - RESTful な URL とアクションのセットを自動で生成する Scaffold でも利用される
- **HTTP ステータスコードの指定:**
  - `render 'new', status: :unprocessable_entity` (または `status: 422`): バリデーションエラーなどでフォームを再表示する際に使用
  - `redirect_to root_url, status: :see_other` (または `status: 303`): DELETE リクエスト後など、リダイレクトを行う際に Turbo と連携して正しく動作させるために使用 Rails 7.1 以降の Scaffold ではデフォルトで付与される
- **メソッドの置き場所の考え方 (ヘルパー vs モデル):**
  - **ヘルパーメソッド (`app/helpers/`):**
    - 主にビューの表示ロジックを整理・簡略化するために使用
    - データの整形、HTML 生成の補助、複数のビューやコントローラで共通して利用する表示関連の処理やセッション管理など、横断的な関心事に向く
  - **モデル内メソッド (`app/models/`):**
    - そのモデルのデータに密接に関連するビジネスロジックやデータ操作、データ固有の振る舞いを記述
    - データのバリデーション、データの作成・更新・削除、関連データの取得、特定のモデルインスタンスに対する操作など

### ビュー・フォーム関連

- **`form_with` ヘルパーメソッドのオプション:**
  1.  **`model: @object`**:
      - **用途**: Active Record のモデルオブジェクト (`@user` など) に基づいてフォームを作成・更新する場合
      - **URL**: Rails がモデルから自動推測 (例: `@user` が新規オブジェクトなら `users_path`、既存オブジェクトなら `user_path(@user)`)
      - **パラメータ**: `params[:モデル名]` (例: `params[:user]`) で受け取る
  2.  **`url: path, scope: :symbol`**:
      - **用途**: モデルオブジェクトに直接紐付かないフォーム (ログイン、検索、パスワード再設定リクエストなど)
      - **URL**: `url`オプションで明示的に指定
      - **パラメータ**: `scope`オプションで指定したシンボルをキーとして `params[:シンボル名]` (例: `scope: :session` なら `params[:session]`) で受け取る
- **`attr_accessor`:**
  - クラスのインスタンス変数に対するゲッターメソッドとセッターメソッドを自動で定義する
  - データベースに保存しない一時的な属性（仮想属性）をモデルで扱いたい場合などによく使われる
  - 語源: `attr` (attribute: 属性) + `accessor` (access する装置/人)
- **`flash.now`:**
  - `render` で同じページを再描画する際にフラッシュメッセージを表示したい場合に使用 通常のリクエストをまたがない
- **`pluralize` ヘルパー:**
  - 数値に応じて名詞の単数形・複数形を適切に表示してくれる 英語の文法ミスを防ぐのに便利
- **`escape` (`CGI.escape`)**:
  - URL に含める文字列（特にメールアドレスなど、`@` や `.` を含むもの）を安全な形式にエンコード（エスケープ）する メール内のリンクをテストする際などに、エスケープされた文字列と照合するのに使う
- **`store_location` メソッド:**
  - ログイン前にアクセスしようとしていた URL をセッションに保存しておき、ログイン後にその URL にリダイレクトする「フレンドリーフォワーディング」を実現するために使われる `store` は「保存する、蓄える」の意
- **`<li id="micropost-<%= micropost.id %>">` のような ID 指定:**
  - 各要素に一意な ID を付与することで、CSS でのスタイリングや JavaScript での DOM 操作が容易になる
- **`request.referrer` メソッド:**
  - 一つ前にいたページの URL（リファラ）を取得できる マイクロポスト削除後など、元のページに戻りたい場合に便利

#### JavaScript (Importmap)

- **Importmap による JavaScript 導入の流れ:**
  1.  **前提確認:**
      - `Gemfile` に `importmap-rails` があるか
      - レイアウト (`application.html.erb`) に `javascript_importmap_tags` があるか
      - `app/assets/config/manifest.js` に JS ディレクトリへのリンク (`//= link_tree ../../javascript .js` など) があるか
  2.  **JS ファイル作成:**
      - `app/javascript/` 配下 (例: `app/javascript/custom/`) に JS ファイル (例: `menu.js`) を作成
  3.  **Importmap で登録 (`config/importmap.rb`):**
      - ディレクトリごと: `pin_all_from "app/javascript/custom", under: "custom"`
      - ファイルごと: `pin "my_module", to: "my_module.js"`
  4.  **`application.js`からインポート (`app/javascript/application.js`):**
      - 例: `import "custom/menu"` (上記`pin_all_from`の場合)
      - 例: `import "my_module"` (上記`pin`の場合)

#### Hotwire (Turbo Stream)でページ部分更新

- **処理の流れ**
  1.  ボタンクリック → Turbo Stream リクエスト送信
  2.  コントローラ: `respond_to` で `format.turbo_stream` を実行
  3.  Rails: 対応する `アクション名.turbo_stream.erb` をレンダリング
  4.  ブラウザ: `turbo_stream.update` の指示で指定 ID の要素内容を差し替え (ページ再読み込みなし)
- Hotwire を使う場合のテストは`format: :turbo_stream`オプションを指定
- **導入の流れ:**

  1.  **コントローラ**: `respond_to` で Turbo Stream リクエストに対応

      - `format.turbo_stream` を追加
      - Turbo Stream テンプレートで使うデータはインスタンス変数 (`@user`) で渡す

      ```ruby
      # app/controllers/relationships_controller.rb
      def create
        @user = User.find(params[:followed_id])
        current_user.follow(@user)
        respond_to do |format|
          format.html { redirect_to @user }
          format.turbo_stream # Turbo Streamリクエスト時の処理
        end
      end
      ```

  2.  **ビュー**: 更新したい HTML 要素に CSS ID を付与

      - 例: `<div id="follow_form">...</div>`
      - 例: `<strong id="followers" class="stat">...</strong>`

  3.  **Turbo Stream テンプレート作成 (`app/views/relationships/アクション名.turbo_stream.erb`)**:

      - `turbo_stream.update "CSSのID" do ... end` で要素を置き換え
      - ブロック内には新しい HTML（パーシャルを render することが多い）を記述

      ```erb
      <%# create.turbo_stream.erb %>
      <%= turbo_stream.update "follow_form" do %>
        <%= render partial: "users/unfollow" %>
      <% end %>
      <%= turbo_stream.update "followers" do %>
        <%= @user.followers.count %>
      <% end %>
      ```

### テスト (Minitest)

- **`minitest/reporters`:**
  - Minitest のテスト結果の表示を RSpec のように見やすく整形する gem
- **テスト駆動開発 (TDD) のリズム:**
  - Red (失敗するテストを書く) → Green (テストをパスする最小限のコードを書く) → Refactor (コードを綺麗にする) のサイクル
- **統合テスト (Integration Test):**
  - 複数のコンポーネント (コントローラ、ビュー、モデルなど) が連携して正しく動作するかをテストする
  - Rails の統合テストは、HTTP リクエストレベルでの連携をシミュレートする
  - RSpec のシステムスペックや Rails のシステムテストは、Capybara などを使って実際のブラウザ操作をシミュレートする、よりエンドユーザーに近い E2E (End-to-End) テスト
- **Fixture での ERB 利用:**
  - `test/fixtures/*.yml` ファイル内では ERB (`<%= ... %>`) を使用して動的にテストデータを生成できる
- **`assert`系メソッドのニュアンス:**
  - Minitest の `assert_` で始まるメソッドは「～であることを表明する」「～であるべきだ」というニュアンスが強い
    - `assert_equal(expected, actual)`: `expected` (期待する値) と `actual` (実際の値) が等しいことを表明
    - `assert(expression)`: `expression` (式) が真 (true) であることを表明
    - `assert_not(expression)`: `expression` が偽 (false) であることを表明
    - `assert_nil(object)`: `object` が `nil` であることを表明
    - `assert_redirected_to(path)`: 指定された `path` にリダイレクトされたことを表明
    - `assert_select(selector, ...)`: HTML 内に指定された `selector` に一致する要素が存在し、内容や属性が期待通りであることを表明
  - RSpec の `expect(actual).to eq(expected)` のように「期待」を前面に出す書き方とは少し異なる
- **ページネーション (`will_paginate`):**
  - `will_paginate` と `bootstrap-will_paginate` gem を利用
  - ビューに `<%= will_paginate @collection %>` のように記述
  - コントローラのアクションで `@collection = Model.paginate(page: params[:page])` のようにしてデータを取得

## III. セキュリティ関連

- **セッション管理と Cookie:**
  - Rails の `session` メソッドで作成される Cookie は、通常ブラウザを閉じると有効期限が切れる一時的なもの
  - ただし、ブラウザの「閉じたときの状態を復元する」機能が有効だと、セッション Cookie も復元されてしまうことがある これは Rails 側では制御できないブラウザの挙動
- **メタプログラミングと `send` メソッド:**
  - `send` メソッドは、オブジェクトに対してメソッド名を文字列やシンボルで動的に指定して呼び出すことができる Ruby の強力な機能
  - DRY 原則を実践するためや、柔軟なコードを書くために使われることがある（例: `authenticated?` メソッドの実装）

## IV. 後で詳しく調べたいことメモ

- アセットパイプラインという言葉そのものについて要再確認
- パーシャルは自動生成せずに、エディタを使って手動で作成するのが一般的なのは何故か？ (Scaffold 以外の場合)
- Active Record に対応する SQL コマンド (具体的にどのような SQL が発行されているか)
- TDD の Red/Green のリズムは何故
- `add_index` の詳細な効果とユースケース
- `has_secure_password` の内部実装の詳細
- Sass のミックスイン（mixin）機能
- `form_with` 苦手
- セッションには Session モデルがない & Active Record モデルではない
- Bootstrap の命名規則、グリッドシステム
- コントローラ用ヘルパーとモデル用`concern`の使い分け
- `digest`メソッド
- ラムダ (`->`)、Proc オブジェクト
- `request.referrer` メソッド、フレンドリーフォワーディング以外いつ使うのか
- Hotwire
- SQL 思い出し
