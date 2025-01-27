---
title: "Solid Queueを使ってみた"
emoji: "🗒️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rails", "ruby", "resque", "solidqueue"]
published: true
---

本記事では、Rails 8 から導入された Solid Queue について紹介します。

## Solid Queueとは

ActiveJob において、キューイングシステムとして今回試験的に Solid Queue を導入します。

Solid Queue は通常のデータベースを利用する Active Job 用のキューイングシステムとなってます。

本来、Resque などのキューイングシステムを使う場合 Redis などのインメモリデータストア等が必要になります。

しかし、Redis は準備が面倒なのと、今回のサンプルではそこまで使う必要もないです。

そして、DB に「SQLite」を利用しているので「Solid Queue」を使うこととします。

## Solid Queueの準備

Solid Queue のインストールと導入コマンドは以下です。

```zsh
$ bundle add solid_queue
$ bin/rails solid_queue:install
```

上記を実行することで必要なファイルの追加・更新が発生します。

`development.rb`に以下を追記します。

```rb
  # Use Solid Queue in Development.
  config.active_job.queue_adapter = :solid_queue
  config.solid_queue.connects_to = { database: { writing: :primary } }
```

こちらの設定は、`bin/rails solid_queue:install`を実行した際に、`production.rb`に追記されたものを転記する形となります。

`writing`の設定は`database.yml`と関連付けです。

また development 環境では、Solid Queue 用の別 DB は使わず既存の SQLite DB を利用するよう設定変更します。

1. `db/queue_schema.rb` が追加されていたはずなので、そのファイルの内容で migration ファイルを作成
2. `db/queue_schema.rb` は削除
3. `rails db:migrate` で既存 DB に Solid Queue 関連テーブル作成
4. `config.solid_queue.connects_to`の設定は`production.rb`から削除

本来 Solid Queue は `./bin/jobs`で起動しますが、development 環境では`puma`のプラグインとして起動させます。

`puma.rb`に以下を追加します。

```rb
plugin :solid_queue if ENV["SOLID_QUEUE_IN_PUMA"] || Rails.env.development?
```


もし DB を分けたい場合、以下のようにすることで別 DB に queue を動かすことができるようになります。

```rb
development:
+ primary:
    <<: *default
    database: storage/development.sqlite3
+  queue:
+    <<: *default
+    database: storage/development_queue.sqlite3
+    migrations_paths: db/queue_migrate
```

今回は単一 DB で動くように設定しておきました。

```rb
development:
  primary: &development_primary
    <<: *default
    database: storage/development.sqlite3
```

`config.solid_queue.connects_to = { database: { writing: :primary } }`で設定した`writing: :primary`について。

`database.yml`の `:primary` と命名を一致させておく必要があります。

## 定期実行タスクについて

Solid Queue には定期実行タスクを定義可能です。

これは、Resque にある Resque Schedule と同じですね。

試しに job を作って動かしてみましょう。

### HelloSampleJobの作成


```zsh
$ rails generate job hello_sample_job
      invoke  test_unit
      create    test/jobs/hello_sample_job_test.rb
      create  app/jobs/hello_sample_job.rb
```

JOB ファイルを以下のように記載します。

```rb
class HelloSampleJob < ApplicationJob
  queue_as :default

  def perform(*args)
    pp args
  end
end
```

### 定期実行定義

定期実行タスク定義は、先ほど生成した `config/recurring.yml`です。

```yml
# production:
#   periodic_cleanup:
#     class: CleanSoftDeletedRecordsJob
#     queue: background
#     args: [ 1000, { batch_size: 500 } ]
#     schedule: every hour
#   periodic_command:
#     command: "SoftDeletedRecord.due.delete_all"
#     priority: 2
#     schedule: at 5am every day
```

今はこのように全てコメントアウトされた状態となってますがサンプルに合わせ以下のように修正します。

```yml
development:
  hello_sample_job:
    class: HelloSampleJob
    args: [ 42, { status: "custom_status" } ]
    schedule: every second
```

こうすることで定期的な JOB 実行が可能となります。

また、別ファイル名(`schedule.yml`)で指定したい場合、`--recurring_schedule_file`をつけることで対応可能です。

```zsh
bin/jobs --recurring_schedule_file=config/schedule.yml
```

## 管理画面を用意する

Solid Queue の状態を確認するための管理画面を導入します。

Resque には、resque-scheduler を使えば管理画面が用意されています。

Solid Queue の場合ですが、README に以下記載があります。


>However, we recommend taking a look at mission_control-jobs. This dashboard allows you to examine, retry and discard failed jobs.

とのことで、mission_control-jobs を使うことにします。

### mission_control-jobs install

Gemfile に追記して install します。

```
gem "mission_control-jobs"
```

```
$ bundle install
```

route を以下で定義します。

```rb
Rails.application.routes.draw do
  # ...
  mount MissionControl::Jobs::Engine, at: "/jobs"
```

ただしこれだけではアクセスできず、認証をセットアップする必要があります。

```
$ bin/rails mission_control:jobs:authentication:configure
```

もし認証不要とする場合、`application.rb`に以下を追記すれば良いでしょう。

```rb
config.mission_control.jobs.http_basic_auth_enabled = false
```

ここまでやると、以下画面に`/jobs`でアクセスできるはずです。

![mission_control](/images/mission_control.png)

以下画面で JOB を実行させることが可能です。

![mission_control_job1](/images/mission_control_job1.png)


実行ボタンを押すと以下画面になります。

![mission_control_job2](/images/mission_control_job2.png)

そして実行中の時はこのようになります。

![mission_control_job3](/images/mission_control_job3.png)


## まとめ

以上が Solid Queue を使った ActiveJob の設定となってます。


本番運用ではまた別途考慮するべきところは多くあるものの、Local 環境で手軽に Job が使えること・個人開発レベルでもお手軽に導入できそうです。
