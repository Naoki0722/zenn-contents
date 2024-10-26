---
title: "Railsの基本的な設定(Hotwire手前まで)"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rails", "ruby"]
published: false
publication_name: "yamaden"
---

ここからは、Hotwire の基本的な使い方を紹介します。

Rails のインストールについては下記記事でまとめました。


[Rails+Hotwireの環境構築(./bin/devまで)](./rails_hw_install)


前回まではただ単にインストールしただけなので、どのようなのを作りたいかを考えておきます。

今回は DM 機能をベースに作ります。

- User
- Room
- Message

Room に受信者 ID と送信者 ID の user 情報を持たせることで、room でどっちが送信者か受信者かを判別できるようにします。

さらに条件として、受信者 ID と送信者 ID の複合ユニークとして重複した room が作られないよう制限します。

また、Message には room_id と user_id を持たせることでどのルームのどっちのユーザーが送信したかを判別できるようにします。


## やること

1. devise のインストール
2. テーブル・モデルの作成
3. Tailwind の設定
4. Controller・View の作成
5. Hotwire へ技術置き換え
6. Actinon Cable を利用したリアルタイム通信

ひとまず本記事では、1~4 までを普通の Rails で作成します。

そして、その後別記事にて Hotwire の仕組みに置き換えます。

さらに別記事になりますが、Action Cable を利用したリアルタイム通信処理まで実装する予定です。

また DM 機能にはには既読機能もつけたいですが、今回のスコープからは外します。


## deviseのインストール

テーブルを作ってからインストールしても良かったのですが、User を認証モデルとして利用したいので、先にインストールしちゃいます。

[devise github](https://github.com/heartcombo/devise)


まずは下記コマンドを実行し、Gemfile への追加・install 処理をします。

```zsh
$ bundle add devise
$ rails generate devise:install

      create  config/initializers/devise.rb
      create  config/locales/devise.en.yml
===============================================================================

Depending on your application's configuration some manual setup may be required:

  1. Ensure you have defined default url options in your environments files. Here
     is an example of default_url_options appropriate for a development environment
     in config/environments/development.rb:

       config.action_mailer.default_url_options = { host: 'localhost', port: 3000 }

     In production, :host should be set to the actual host of your application.

     * Required for all applications. *

  2. Ensure you have defined root_url to *something* in your config/routes.rb.
     For example:

       root to: "home#index"

     * Not required for API-only Applications *

  3. Ensure you have flash messages in app/views/layouts/application.html.erb.
     For example:

       <p class="notice"><%= notice %></p>
       <p class="alert"><%= alert %></p>

     * Not required for API-only Applications *

  4. You can copy Devise views (for customization) to your app by running:

       rails g devise:views

     * Not required *

===============================================================================
```

色々とコメントが流れてますが以下のことが書かれています。

- mailer を使う場合の url 指定が可能
- routes.rb で root_url を指定すること
- layouts/application.html.erb に Flash メッセージがあること
- devise の view をカスタマイズしたい場合は、`rails g devise:views`を実行すること


続いて User モデルに Devise 機能を紐づけます。

```zsh
$ rails generate devise MODEL

      invoke  active_record
      create    db/migrate/20241019014656_devise_create_users.rb
      create    app/models/user.rb
      invoke    test_unit
      create      test/models/user_test.rb
      create      test/fixtures/users.yml
      insert    app/models/user.rb
       route  devise_for :users
```

するとマイグレーションファイルや User モデルなどが生成します。
また、`config/routes.rb`に `devise_for :users`が追加されてます。


User.rb を覗いてみましょう。

```rb
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable
end
```

devise に以下シンボルが渡されています。

- database_authenticatable
  - DB に保存するパスワードの暗号化(必須)
- registerable
  - 登録処理
- recoverable
  - パスワードリセット
- rememberable
  - Cookie にログイン情報を保持
- validatable
  - E メールとパスワードのバリデーション

この状態で 1 回 migrate しましょう。

```zsh
$ rails db:migrate
```


一旦インストールは完成ですが、サインインやサインアップ処理等をすると root_path にリダイレクトします。

例えば、:user リソースを使用する場合、user_root_path が存在すればそれが使用されますが、そうでなければデフォルトの root_path が使用されます。

つまり devise 処理後の root_path を設定しておく必要があるため、設定します。


```zsh
$ rails g controller HomeController index

      create  app/controllers/home_controller.rb
       route  get "home/index"
      invoke  tailwindcss
      create    app/views/home
      create    app/views/home/index.html.erb
      invoke  test_unit
      create    test/controllers/home_controller_test.rb
      invoke  helper
      create    app/helpers/home_helper.rb
      invoke    test_unit
```

そして `routes.rb`に以下追記します。

```rb
root "home/index"
```

あとは、実際に`/users/sign_up`や`/users/sign_in`にアクセスすると devise が用意してくれているフォーム画面に遷移して実際に登録等が可能となります。

とりあえず devise 導入は一旦ここまでにします。

## テーブル・モデルの作成

続いて関連するテーブルやモデルを追加していきます。

DM 機能としてシンプルな設計を目指します。

- User
- Room
- Message

User はすでに存在するため追加するカラムや関連テーブルのリレーションを設定します。

必要な項目の洗い出しを実施します。

User
- email
- password
- name

Room
- creator_id
- recipient_id

(creator_id+recipient_id の複合ユニークキー)

Message
- content
- user_id
- room_id


一旦以下の通りテーブルを作成しました。

```rb
class AddNameToUsers < ActiveRecord::Migration[7.2]
  def change
    add_column :users, :name, :string, null: false, default: ""
  end
end
```



```rb
class CreateRooms < ActiveRecord::Migration[7.2]
  def change
    create_table :rooms do |t|
      t.references :creator, null: false, foreign_key: { to_table: :users }
      t.references :recipient, null: false, foreign_key: { to_table: :users }

      t.timestamps
    end
    add_index :rooms, [ :creator_id, :recipient_id ], unique: true
  end
end
```

```rb
class CreateMessages < ActiveRecord::Migration[7.2]
  def change
    create_table :messages do |t|
      t.text :content
      t.references :user, null: false, foreign_key: true
      t.references :room, null: false, foreign_key: true

      t.timestamps
    end
  end
end
```

migration します。

```zsh
$ rails db:migrate

== 20241019070843 AddNameToUsers: migrating ===================================
-- add_column(:users, :name, :string, {:null=>false, :default=>""})
   -> 0.0008s
== 20241019070843 AddNameToUsers: migrated (0.0009s) ==========================

== 20241019071050 CreateRooms: migrating ======================================
-- create_table(:rooms)
   -> 0.0009s
== 20241019071050 CreateRooms: migrated (0.0009s) =============================

== 20241019071240 CreateMessages: migrating ===================================
-- create_table(:messages)
   -> 0.0016s
== 20241019071240 CreateMessages: migrated (0.0017s) ==========================
```

テーブル作成は完了し、model の関連づけを進めます。

とはいっても、generation コマンドで model 生成した段階である程度のリレーションはできているので細かな設定をしていきます。

user.rb

```rb
# frozen_string_literal: true

class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :timeoutable, :trackable and :omniauthable
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable

  has_many :messages
  has_many :sent_rooms, class_name: "Room", foreign_key: "creator_id"
  has_many :received_rooms, class_name: "Room", foreign_key: "recipient_id"
end

```

room.rb

```rb
# frozen_string_literal: true

class Room < ApplicationRecord
  belongs_to :creator, class_name: "User", foreign_key: "creator_id"
  belongs_to :recipient, class_name: "User", foreign_key: "recipient_id"
  has_many :messages, dependent: :destroy

  validates :creator_id, uniqueness: { scope: :recipient_id }

  before_validation :normalize_user_order

  private

  def normalize_user_order
    if creator_id > recipient_id
      self.creator_id, self.recipient_id = recipient_id, creator_id
    end
  end
end

```

message.rb

```rb
# frozen_string_literal: true

class Message < ApplicationRecord
  belongs_to :user
  belongs_to :room
end
```

## Tailwindの設定

認証周り、テーブル周りの準備ができたので、今後は view 周りの設定を進めていきます。

そのために Tailwind が正常に動くよう設定します。

と思いきや、rails new した時に css で tailwind を指定したので特に設定なしで使える状態でした。

application.html.erb

```erb
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for(:title) || "Action Cable Hotwire Rails" %></title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>

    <%= yield :head %>

    <link rel="manifest" href="/manifest.json">
    <link rel="icon" href="/icon.png" type="image/png">
    <link rel="icon" href="/icon.svg" type="image/svg+xml">
    <link rel="apple-touch-icon" href="/icon.png">
    <%= stylesheet_link_tag "tailwind", "inter-font", "data-turbo-track": "reload" %>
    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
  </head>

  <body>
    <main class="container mx-auto mt-28 px-5 flex">
      <%= yield %>
    </main>
  </body>
</html>
```

## Controller・View の作成

いよいよ、デザイン作成を進めていきます。

画面のイメージとしては以下です。

- top ページに DM のルーム一覧へ飛ぶリンクを設置
- DM ルーム一覧のページからそれぞれの DM ルームへ遷移
- チャット画面が表示され、そこでやり取りをする


まずは rooms 一覧、room 詳細画面を作っていきます。

```zsh
$ rails g controller rooms index show

      create  app/controllers/rooms_controller.rb
       route  get "rooms/index"
              get "rooms/show"
      invoke  tailwindcss
       exist    app/views/rooms
      create    app/views/rooms/index.html.erb
      create    app/views/rooms/show.html.erb
      invoke  test_unit
      create    test/controllers/rooms_controller_test.rb
      invoke  helper
      create    app/helpers/rooms_helper.rb
      invoke    test_unit
```

とりあえずトップ画面に遷移リンクを作りましょう。


```erb
<%= link_to 'DMリスト一覧へ', rooms_path %>
```

HOME から`/rooms`へ遷移すること確認しましょう。

実はこの時点で Turbo の機能が動いています。

正確に伝えると Turbo Drive の設定が`app/javascript/application.js`に下記コードが記載されており、すでに機能として動いています。

```js
// Configure your import map in config/importmap.rb. Read more: https://github.com/rails/importmap-rails
import "@hotwired/turbo-rails"
import "controllers"
```


このコードに`Turbo.session.drive = false`を追記することで Turbo の機能しなくなります。

実際に記述後、画面遷移すると画面ロードして遷移することが確認できるはずです。

また、画面でデータをサンプルとして表示するためテストデータを作成します。

console を使ってデータを投入しておきます。

```zsh
$ rails c

User.create!(name:'test', email: 'fsafas@example.com', created_at: Time.zone.now, updated_at:  Time.zone.now, password: '01380722')
User.create!(name:'test_2', email: 'fsafadddds@example.com', created_at: Time.zone.now, updated_at:  Time.zone.now, password: '01380722')

Room.create!(creator_id: 1, recipient_id:2)

Message.create!(content: 'testtesttestです', user_id: 1, room_id:1)

User.create!(name:'田中', email: 'tanaka@example.com', password: '01380722')
```

デザイン詳細はこちらにて。
