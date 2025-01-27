---
title: "Todoアプリ(Turbo Stream)にActionCableを導入"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rails", "ruby", "hotwire"]
published: true
publication_name: "yamaden"
---

以前の記事で、Todo アプリを作成しました。

その際は、Turbo Stream の課題について説明し、複雑性を減らすなら Turbo Drive の morphing を使えば解決できるのではないかという提案をしました。

一方、リアルタイム更新などを実施するには、Turbo Stream を利用しなければ実現不可能です。

本記事では、Turbo Stream を使ったリアルタイム処理ができるように実装していきます。

一旦、前回の Turbo Drive への置き換え commit は revert し、Turbo Stream 利用状態に戻しておきます。


## ActionCableの利用について

本来、ActionCable を使ったリアルタイム通信となるとチャネルの用意、サブスクリプションの設定など色々なファイルを追加することとなります。

しかし、Hotwire を使うと「Turbo::StreamsChannel」が用意されているのでこちらを使ってシンプルに実装できます。

## Solid Cableについて

Rails には「Solid Cable」というのがあります。

Solid Cable とは、Action Cable サブスクリプションアダプタでデータベースを用いる方法をとってます。

本来、Action Cable を使う際、サブスクリプションアダプタとしてインメモリデータベースの「Redis」を使うケースが一般的です。

その設定は、`config/cable.yml`に定義されてます。

```yml
development:
  adapter: async

test:
  adapter: test

production:
  adapter: redis
  url: <%= ENV.fetch("REDIS_URL") { "redis://localhost:6379/1" } %>
  channel_prefix: todo_morph_production
```

development 環境などでは準備不要とするため、async アダプタが設定されてます。

しかし、development 環境や test 環境で利用するためのものなので、production 環境では使わないでください。

Redis の環境構築が大変であったりするため、そこで登場したのが Solid Cable です。

今回はこれを使ってみます。

補足情報ですが、Solid Cable のパフォーマンスはほとんどの状況で Redis に匹敵するそうです。

### install

```zsh
$ bundle add solid_cable
$ bundle install
$ bin/rails solid_cable:install
      create  db/cable_schema.rb
       force  config/cable.yml
```

`cable.yml`が update されます。

```yml
# Async adapter only works within the same process, so for manually triggering cable updates from a console,
# and seeing results in the browser, you must do so from the web console (running inside the dev process),
# not a terminal started via bin/rails console! Add "console" to any action or any ERB template view
# to make the web console appear.
development:
  adapter: async

test:
  adapter: test

production:
  adapter: solid_cable
  connects_to:
    database:
      writing: cable
  polling_interval: 0.1.seconds
  message_retention: 1.day
```

また、`db/cable_schema.rb`というのが生成します。


Solid Cable を使用する場合、`config/cable.yml` ファイルで `connects_to` ブロックの設定をすれば良いです。

その際、`config/cable.yml`でデータベースに使用する名前と `config/database.yml` で定義した名前が一致させなければなりません。


本来は下記のように Solid Cable 用で DB を分ける設定となります。

```rb
development:
  primary:
    <<: *default
    database: storage/production.sqlite3
  cable:
    <<: *default
    database: storage/production_cable.sqlite3
    migrations_paths: db/cable_migrate
```
しかし、
solid_cable で使う DB をアプリケーション DB と同じにする場合はまとめて OK です。


config/database.yml

```rb
development:
  primary:
    <<: *default
    database: storage/production.sqlite3
```

config/cable.yml

```yml
development:
  adapter: solid_cable
  connects_to:
    database:
      writing: primary
  polling_interval: 0.1.seconds
  message_retention: 1.day
```

- また、db/cable_schema.rb を migration ファイルに作成
- db/cable_schema.rb を削除
- bin/rails db:migrate を実行


ここまで実行すれば Solid Cable の設定は OK です。

## Hotwire + ActionCableの設定

ここからは Hotwire の Turbo Stream で Action Cable を使ったリアルタイム更新ができるように設定していきます。


Action Cable 経由で更新したいページには、turbo_stream_from メソッドでストリーム名を指定します。ここでは 'todos' としています。


```erb
<%= turbo_stream_from 'todos' %>
<div class="max-w-4xl mx-auto px-4 py-8">
  <h1 class="text-3xl font-bold mb-8">Todos</h1>
.
.
.
```

まずは update 時の`update.turbo_stream.erb`の以下を削除し、stream 経由を削除します。

```erb
<%= turbo_stream.replace @todo %>
```

逆に controller 側は、broadcast で配信するように設定します。

ここでは、`broadcast_update_to`を使ってます。

```rb
  def update
    unless @todo.update(todo_params)
      render :edit, status: :unprocessable_entity
    end
    @todo.broadcast_update_to "todos"
  end
```

本来は以下のように設定するのですが、省略することで上記のような記述方法になってます。

```rb
clearance.broadcast_update_to examiner.identity, :clearances, partial: "clearances/other_partial", locals: { a: 1 }
```


こうすると更新時に別タブで開いていたものも更新されます。

同じように create と destory も対応していきます。

そうすると controller には以下が追加されることとなりました。

```rb
@todo.broadcast_prepend_to "todos"
@todo.broadcast_update_to "todos"
@todo.broadcast_remove_to "todos"
```

そこで、公式のソースを見にいきます。

https://github.com/hotwired/turbo-rails/blob/main/app/models/concerns/turbo/broadcastable.rb


このようなメソッドがあります。

```rb
    # Configures the model to broadcast creates, updates, and destroys to a stream name derived at runtime by the
    # <tt>stream</tt> symbol invocation. By default, the creates are appended to a dom id target name derived from
    # the model's plural name. The insertion can also be made to be a prepend by overwriting <tt>inserts_by</tt> and
    # the target dom id overwritten by passing <tt>target</tt>. Examples:
    #
    #   class Message < ApplicationRecord
    #     belongs_to :board
    #     broadcasts_to :board
    #   end
    #
    #   class Message < ApplicationRecord
    #     belongs_to :board
    #     broadcasts_to ->(message) { [ message.board, :messages ] }, inserts_by: :prepend, target: "board_messages"
    #   end
    #
    #   class Message < ApplicationRecord
    #     belongs_to :board
    #     broadcasts_to ->(message) { [ message.board, :messages ] }, partial: "messages/custom_message"
    #   end
    def broadcasts_to(stream, inserts_by: :append, target: broadcast_target_default, **rendering)
      after_create_commit  -> { broadcast_action_later_to(stream.try(:call, self) || send(stream), action: inserts_by, target: target.try(:call, self) || target, **rendering) }
      after_update_commit  -> { broadcast_replace_later_to(stream.try(:call, self) || send(stream), **rendering) }
      after_destroy_commit -> { broadcast_remove_to(stream.try(:call, self) || send(stream)) }
    end
```

つまり、Model に`broadcasts_to`を定義すれば、一行で表現が可能ということです。

早速、controller の記述を削除し、`todo.rb`に定義し直します。

```rb
broadcasts_to -> (_todo) { "todos" }, inserts_by: :prepend
```

これで Turbo Stream と Action Cable を使ったリアルタイム更新が実装できました。
