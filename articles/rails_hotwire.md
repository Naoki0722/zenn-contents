---
title: "Rails+Hotwireの環境構築"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rails", "ruby"]
published: true
publication_name: "yamaden"
---

この記事では一から Rails の環境構築と Hotwire を導入した流れを記載していきます。


## Railsのインストール環境

Rails のインストールから対応していきます。

ですが、まずここでどのような構成にするか決める必要があります。

- 実行環境
  - Local に環境構築するか
  - Docker に環境構築するか
- DB
  - MySQL を使うか
  - SQLite を使うか
- CSS
  - Bootstrap を使うか
  - Bulma を使うか
  - TailwindCSS を使うか

結論ですが、下記で進めます。

今回の作業は最適な環境構築をすることではなく、Hotwire を使うことが目的です。

ですので、私が普段慣れ親しんでる技術 かつ シンプルに取り組むこととしました。

- Rails 7.2
- Ruby 3.3.5
- TailwindCss
- SQLite


## アセットパイプラインについて

Rails には、「アセットパイプライン」という仕組みがあります。

細かな説明は省きますが、うまいこと JavaScript と CSS アセットの配信をしてくれるものです。

アセットパイプラインとしては、以下のようなものがあります。

- sprockets
- propshaft

デフォルトインストールでは、`sprockets`が選択されるはずです。

どちらがいいというのは利用プロジェクトによります。

両者の違いとしては、トランスパイル、圧縮・連結などの処理をするかしないかになります。

`propshaft` は、トランスパイル、圧縮・連結などの処理をせず、ダイジェストの追加だけ実行するため、`sprockets`よりも高速で動作するのが特徴です。


また JavaScript の配信のため、以下ツールを使います。

- importmap-rails
- jsbundling-rails
  - importmap-rails 方式の代わりに以下のいずれかを JavaScript のバンドルに利用できます
    - Bun
    - esbuild
    - rollup.js
    - Webpack
  - JavaScript コードがトランスパイルに依存している場合選択した方がいい

デフォルトインストールでは、`importmap-rails`が選択されるはずです。


importmap-rails では、CSS の配信には利用できません。

css 配信のため、以下ツールが存在します。

- cssbundling-rails
  - アセットパイプライン経由で CSS を配信するためのツール

Rails 7 以降のデフォルト設定では、以下のツールが使用されています。

sprockets: シンプルなアセットパイプラインで、アセットの結合や圧縮。
importmap-rails: JavaScript アセットパイプラインとして利用。


ドキュメントではこのように記載があります。

### 選定について

>cssbundling-railsはCSSの処理をNode.jsに依存しています。 dartsass-rails gemとtailwindcss-rails gemは、それぞれTailwind CSSとDart Sassのスタンドアロン版実行ファイルを使うので、Node.jsに依存しません。 JavaScriptをimportmap-railsで処理し、CSSをdartsass-railsまたはtailwindcss-railsで処理する形にすれば、Node依存を完全に避けられるので、よりシンプルなソリューションとなります。

css をトランスパイルするとかなら、`cssbundling-rails`を使う選択はありです。

しかし、今回のプロジェクトのように Tailwind を使いたいなら、tailwindcss-rails gem を使えば Node が必要にならなくなるのです。

## インストール作業

さて、ややこしいアセットパイプラインの選定が終わったところで、実際に Rails アプリケーションをインストールしていきます。


とりあえず`rails new`で作成できるのですが、コマンドを見てみましょう。

```zsh
$ rails new -h

Usage:
  rails new APP_PATH [options]

Options:
                 [--skip-namespace]                                            # Skip namespace (affects only isolated engines)
                                                                               # Default: false
                 [--skip-collision-check]                                      # Skip collision check
                                                                               # Default: false
  -r,            [--ruby=PATH]                                                 # Path to the Ruby binary of your choice
                                                                               # Default: /Users/naoki/.rbenv/versions/3.3.5/bin/ruby
  -n,            [--name=NAME]                                                 # Name of the app
  -m,            [--template=TEMPLATE]                                         # Path to some application template (can be a filesystem path or URL)
  -d,            [--database=DATABASE]                                         # Preconfigure for selected database
                                                                               # Default: sqlite3
                                                                               # Possible values: mysql, trilogy, postgresql, sqlite3
  -G,            [--skip-git]                                                  # Skip git init, .gitignore and .gitattributes
                 [--skip-docker]                                               # Skip Dockerfile, .dockerignore and bin/docker-entrypoint
                 [--skip-keeps]                                                # Skip source control .keep files
  -M,            [--skip-action-mailer]                                        # Skip Action Mailer files
                 [--skip-action-mailbox]                                       # Skip Action Mailbox gem
                 [--skip-action-text]                                          # Skip Action Text gem
  -O,            [--skip-active-record]                                        # Skip Active Record files
                 [--skip-active-job]                                           # Skip Active Job
                 [--skip-active-storage]                                       # Skip Active Storage files
  -C,            [--skip-action-cable]                                         # Skip Action Cable files
  -A,            [--skip-asset-pipeline]                                       # Indicates when to generate skip asset pipeline
  -a,            [--asset-pipeline=ASSET_PIPELINE]                             # Choose your asset pipeline
                                                                               # Default: sprockets
                                                                               # Possible values: none, sprockets, propshaft
  -J, --skip-js, [--skip-javascript]                                           # Skip JavaScript files
                 [--skip-hotwire]                                              # Skip Hotwire integration
                 [--skip-jbuilder]                                             # Skip jbuilder gem
  -T,            [--skip-test]                                                 # Skip test files
                 [--skip-system-test]                                          # Skip system test files
                 [--skip-bootsnap]                                             # Skip bootsnap gem
                 [--skip-dev-gems]                                             # Skip development gems (e.g., web-console)
                 [--skip-rubocop]                                              # Skip RuboCop setup
                 [--skip-brakeman]                                             # Skip brakeman setup
                 [--skip-ci]                                                   # Skip GitHub CI files
                 [--dev], [--no-dev], [--skip-dev]                             # Set up the application with Gemfile pointing to your Rails checkout
                 [--devcontainer], [--no-devcontainer], [--skip-devcontainer]  # Generate devcontainer files
                                                                               # Default: false
                 [--edge], [--no-edge], [--skip-edge]                          # Set up the application with a Gemfile pointing to the 7-2-stable branch on the Rails repository
  --master,      [--main], [--no-main], [--skip-main]                          # Set up the application with Gemfile pointing to Rails repository main branch
                 [--rc=RC]                                                     # Path to file containing extra configuration options for rails command
                 [--no-rc]                                                     # Skip loading of extra configuration options from .railsrc file
                 [--api], [--no-api], [--skip-api]                             # Preconfigure smaller stack for API only apps
                                                                               # Default: false
                 [--minimal], [--no-minimal], [--skip-minimal]                 # Preconfigure a minimal rails app
  -j, --js,      [--javascript=JAVASCRIPT]                                     # Choose JavaScript approach
                                                                               # Default: importmap
                                                                               # Possible values: importmap, bun, webpack, esbuild, rollup
  -c,            [--css=CSS]                                                   # Choose CSS processor. Check https://github.com/rails/cssbundling-rails for more options
                                                                               # Possible values: tailwind, bootstrap, bulma, postcss, sass
  -B,            [--skip-bundle]                                               # Don't run bundle install
                 [--skip-decrypted-diffs]                                      # Don't configure git to show decrypted diffs of encrypted credentials

Runtime options:
  -f, [--force]                                      # Overwrite files that already exist
  -p, [--pretend], [--no-pretend], [--skip-pretend]  # Run but do not make any changes
  -q, [--quiet], [--no-quiet], [--skip-quiet]        # Suppress status output
  -s, [--skip], [--no-skip], [--skip-skip]           # Skip files that already exist

Rails options:
  -h, [--help], [--no-help], [--skip-help]           # Show this help message and quit
  -v, [--version], [--no-version], [--skip-version]  # Show Rails version number and quit

Description:
    The `rails new` command creates a new Rails application with a default
    directory structure and configuration at the path you specify.

    You can specify extra command-line arguments to be used every time
    `rails new` runs in the .railsrc configuration file in your home directory,
    or in $XDG_CONFIG_HOME/rails/railsrc if XDG_CONFIG_HOME is set.

    Note that the arguments specified in the .railsrc file don't affect the
    default values shown above in this help message.

    You can specify which version to use when creating a new rails application
    using `rails _<version>_ new`.

Examples:
    `rails new ~/Code/Ruby/weblog`

    This generates a new Rails app in ~/Code/Ruby/weblog.

    `rails _<version>_ new weblog`

    This generates a new Rails app with the provided version in ./weblog.

    `rails new weblog --api`

    This generates a new Rails app in API mode in ./weblog.

    `rails new weblog --skip-action-mailer`

    This generates a new Rails app without Action Mailer in ./weblog.
    Any part of Rails can be skipped during app generation.
```


結構オプションは多いですね。

ここで重要なオプションだけピックアップしてみていきます。

### -a[--asset-pipeline=ASSET_PIPELINE]

アセットパイプラインの設定になります。

デフォルトは `sprockets`ですが、そのほかにも `propshaft`が選択可能です。

こちらそのままで問題ありません。

### -d[--database=DATABASE]

データベースの選定になります。

デフォルトは、`sqlite3`でそのほかに `mysql`, `trilogy`, `postgresql`が選択可能です。

今回は SQLite を使うため、オプション指定は不要です。

### -j,--js[--JavaScript=JAVASCRIPT]

JavaScript の配信設定になります。

デフォルトは`importmap`で、そのほかに `bun`, `webpack`, `esbuild`, `rollup`が選択可能です。

JS は、importmap-rails を使いたいのでオプション指定は不要です。

### -c[--css=CSS]

css の設定です。

選択できるオプションは以下です。

- `tailwind`
- `bootstrap`
- `bulma`
- `postcss`
- `sass`

css は今回 tailwind を使いたいので、オプションで`--css=tailwind`とします。

そんなところで、下記コマンドを実行します。

※プロジェクト名を`action_cable_hotwire_rails`にしているのは、この後個人的に action cable を利用したいからです、特に意味はないです。

```zsh
$ rails new action_cable_hotwire_rails --css=tailwind --skip-jbuilder

      create
      create  README.md
      create  Rakefile
      create  .ruby-version
      create  config.ru
      create  .gitignore
      create  .gitattributes
      create  Gemfile
         run  git init from "."
Initialized empty Git repository in /Users/naoki/Documents/matsuzaki-app/action_cable_hotwire_rails/.git/
      create  app
      create  app/assets/config/manifest.js
      create  app/assets/stylesheets/application.css
      create  app/channels/application_cable/channel.rb
      create  app/channels/application_cable/connection.rb
      create  app/controllers/application_controller.rb
      create  app/helpers/application_helper.rb
      create  app/jobs/application_job.rb
      create  app/mailers/application_mailer.rb
      create  app/models/application_record.rb
      create  app/views/layouts/application.html.erb
      create  app/views/layouts/mailer.html.erb
      create  app/views/layouts/mailer.text.erb
      create  app/views/pwa/manifest.json.erb
      create  app/views/pwa/service-worker.js
      create  app/assets/images
      create  app/assets/images/.keep
      create  app/controllers/concerns/.keep
      create  app/models/concerns/.keep
      create  bin
      create  bin/brakeman
      create  bin/rails
      create  bin/rake
      create  bin/rubocop
      create  bin/setup
      create  Dockerfile
      create  .dockerignore
      create  bin/docker-entrypoint
      create  .rubocop.yml
      create  .github/workflows
      create  .github/workflows/ci.yml
      create  .github/dependabot.yml
      create  config
      create  config/routes.rb
      create  config/application.rb
      create  config/environment.rb
      create  config/cable.yml
      create  config/puma.rb
      create  config/storage.yml
      create  config/environments
      create  config/environments/development.rb
      create  config/environments/production.rb
      create  config/environments/test.rb
      create  config/initializers
      create  config/initializers/assets.rb
      create  config/initializers/content_security_policy.rb
      create  config/initializers/cors.rb
      create  config/initializers/filter_parameter_logging.rb
      create  config/initializers/inflections.rb
      create  config/initializers/new_framework_defaults_7_2.rb
      create  config/initializers/permissions_policy.rb
      create  config/locales
      create  config/locales/en.yml
      create  config/master.key
      append  .gitignore
      create  config/boot.rb
      create  config/database.yml
      create  db
      create  db/seeds.rb
      create  lib
      create  lib/tasks
      create  lib/tasks/.keep
      create  lib/assets
      create  lib/assets/.keep
      create  log
      create  log/.keep
      create  public
      create  public/404.html
      create  public/406-unsupported-browser.html
      create  public/422.html
      create  public/500.html
      create  public/icon.png
      create  public/icon.svg
      create  public/robots.txt
      create  tmp
      create  tmp/.keep
      create  tmp/pids
      create  tmp/pids/.keep
      create  vendor
      create  vendor/.keep
      create  test/fixtures/files
      create  test/fixtures/files/.keep
      create  test/controllers
      create  test/controllers/.keep
      create  test/mailers
      create  test/mailers/.keep
      create  test/models
      create  test/models/.keep
      create  test/helpers
      create  test/helpers/.keep
      create  test/integration
      create  test/integration/.keep
      create  test/channels/application_cable/connection_test.rb
      create  test/test_helper.rb
      create  test/system
      create  test/system/.keep
      create  test/application_system_test_case.rb
      create  storage
      create  storage/.keep
      create  tmp/storage
      create  tmp/storage/.keep
      remove  config/initializers/cors.rb
      remove  config/initializers/new_framework_defaults_7_2.rb
         run  bundle install --quiet
         run  bundle lock --add-platform=x86_64-linux
Writing lockfile to /Users/naoki/Documents/matsuzaki-app/action_cable_hotwire_rails/Gemfile.lock
         run  bundle lock --add-platform=aarch64-linux
Writing lockfile to /Users/naoki/Documents/matsuzaki-app/action_cable_hotwire_rails/Gemfile.lock
         run  bundle binstubs bundler
       rails  importmap:install
       apply  /Users/naoki/.rbenv/versions/3.3.5/lib/ruby/gems/3.3.0/gems/importmap-rails-2.0.3/lib/install/install.rb
  Add Importmap include tags in application layout
      insert    app/views/layouts/application.html.erb
  Create application.js module as entrypoint
      create    app/javascript/application.js
  Use vendor/javascript for downloaded pins
      create    vendor/javascript
      create    vendor/javascript/.keep
  Ensure JavaScript files are in the Sprocket manifest
      append    app/assets/config/manifest.js
  Configure importmap paths in config/importmap.rb
      create    config/importmap.rb
  Copying binstub
      create    bin/importmap
         run  bundle install --quiet
       rails  turbo:install stimulus:install
       apply  /Users/naoki/.rbenv/versions/3.3.5/lib/ruby/gems/3.3.0/gems/turbo-rails-2.0.11/lib/install/turbo_with_importmap.rb
  Import Turbo
      append    app/javascript/application.js
  Pin Turbo
      append    config/importmap.rb
         run  bundle install --quiet
       apply  /Users/naoki/.rbenv/versions/3.3.5/lib/ruby/gems/3.3.0/gems/stimulus-rails-1.3.4/lib/install/stimulus_with_importmap.rb
  Create controllers directory
      create    app/javascript/controllers
      create    app/javascript/controllers/index.js
      create    app/javascript/controllers/application.js
      create    app/javascript/controllers/hello_controller.js
  Import Stimulus controllers
      append    app/javascript/application.js
  Pin Stimulus
  Appending: pin "@hotwired/stimulus", to: "stimulus.min.js"
      append    config/importmap.rb
  Appending: pin "@hotwired/stimulus-loading", to: "stimulus-loading.js"
      append    config/importmap.rb
  Pin all controllers
  Appending: pin_all_from "app/javascript/controllers", under: "controllers"
      append    config/importmap.rb
         run  bundle install --quiet
       rails  tailwindcss:install
       apply  /Users/naoki/.rbenv/versions/3.3.5/lib/ruby/gems/3.3.0/gems/tailwindcss-rails-3.0.0/lib/install/tailwindcss.rb
  Add Tailwindcss include tags and container element in application layout
      insert    app/views/layouts/application.html.erb
      insert    app/views/layouts/application.html.erb
      insert    app/views/layouts/application.html.erb
  Build into app/assets/builds
      create    app/assets/builds
      create    app/assets/builds/.keep
      append    app/assets/config/manifest.js
      append    .gitignore
  Add default config/tailwindcss.config.js
      create    config/tailwind.config.js
  Add default app/assets/stylesheets/application.tailwind.css
      create    app/assets/stylesheets/application.tailwind.css
  Add default Procfile.dev
      create    Procfile.dev
  Ensure foreman is installed
         run    gem install foreman from "."
Successfully installed foreman-0.88.1
Parsing documentation for foreman-0.88.1
Done installing documentation for foreman after 0 seconds
1 gem installed
  Add bin/dev to start foreman
      create    bin/dev
  Compile initial Tailwind build
         run    rails tailwindcss:build from "."

Rebuilding...

Done in 212ms.
         run  bundle install --quiet
```

最近の rails はいろんなものを同梱してくれますね。

rubocop の設定や ci の yml、Dockerfile などもデフォルトで入ってきます。

出来上がったプロジェクトの`Gemfile`はこんな感じです。

```gemfile
source "https://rubygems.org"

# Bundle edge Rails instead: gem "rails", github: "rails/rails", branch: "main"
gem "rails", "~> 7.2.1", ">= 7.2.1.1"
# The original asset pipeline for Rails [https://github.com/rails/sprockets-rails]
gem "sprockets-rails"
# Use sqlite3 as the database for Active Record
gem "sqlite3", ">= 1.4"
# Use the Puma web server [https://github.com/puma/puma]
gem "puma", ">= 5.0"
# Use JavaScript with ESM import maps [https://github.com/rails/importmap-rails]
gem "importmap-rails"
# Hotwire's SPA-like page accelerator [https://turbo.hotwired.dev]
gem "turbo-rails"
# Hotwire's modest JavaScript framework [https://stimulus.hotwired.dev]
gem "stimulus-rails"
# Use Tailwind CSS [https://github.com/rails/tailwindcss-rails]
gem "tailwindcss-rails"
# Use Redis adapter to run Action Cable in production
# gem "redis", ">= 4.0.1"

# Use Kredis to get higher-level data types in Redis [https://github.com/rails/kredis]
# gem "kredis"

# Use Active Model has_secure_password [https://guides.rubyonrails.org/active_model_basics.html#securepassword]
# gem "bcrypt", "~> 3.1.7"

# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem "tzinfo-data", platforms: %i[ windows jruby ]

# Reduces boot times through caching; required in config/boot.rb
gem "bootsnap", require: false

# Use Active Storage variants [https://guides.rubyonrails.org/active_storage_overview.html#transforming-images]
# gem "image_processing", "~> 1.2"

group :development, :test do
  # See https://guides.rubyonrails.org/debugging_rails_applications.html#debugging-with-the-debug-gem
  gem "debug", platforms: %i[ mri windows ], require: "debug/prelude"

  # Static analysis for security vulnerabilities [https://brakemanscanner.org/]
  gem "brakeman", require: false

  # Omakase Ruby styling [https://github.com/rails/rubocop-rails-omakase/]
  gem "rubocop-rails-omakase", require: false
end

group :development do
  # Use console on exceptions pages [https://github.com/rails/web-console]
  gem "web-console"
end

group :test do
  # Use system testing [https://guides.rubyonrails.org/testing.html#system-testing]
  gem "capybara"
  gem "selenium-webdriver"
end

```

あとは、画面表示を確認しましょう。

```zsh
$ ./bin/dev
```

この状態で Hotwire もインストールされているので、ここまできたら初期インストールは完了です。
