---
title: "Railsにesbuildを使ってReactを導入する(Dockerで構築)"
emoji: "🏘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react","rails","esbuild"]
published: true
---

最近 Rails+React の構成で業務に当たっており、不明なことがありました。

それは、SPA での構成でなぜそれぞれのアプリを起動せずに SPA が動かせているのかということでした。

個人的にこの辺が全く理解できていないまま、フロントエンドの開発に携わっていたのですが、ゆくゆく調べていくと Rails の上手い使い方があるようです。

Rails には API モードがあり、API サーバーとして機能させることもできるのですが、API モードではなく、Rails の中で React を動かすという方法もできるようです。

今回はその Rails の esbuild の使い方含め、どうやって環境構築をしたかを紹介します。

## 元々RailsはMVCフレームワーク

Rails の全体的な構成としましては、MVC フレームワークであることです。

MVC とは M(データを操作する Model)、V(画面を描画する View)、C(画面とデータの橋渡しの Controller)を使ってアプリケーションを作っていく方法です。

PHP のフレームワークで有名な Laravel もこの MVC を採用しています。

## 近年のクライアントサイドの技術進歩

しかしながら、近年 React や Vue といった JavaScript の技術によって画面側のダイナミックな動きを実現できるようになり、UX の向上や通信負荷の軽減を求めるニーズが高まっています。

Rails もそれに合わせ、クライアント側の開発技術について変遷を遂げてきた過去があるようです。

Rails 3.1 から「アセットパイプライン」というモジュールバンドラーが導入され、クライアントサイド開発をより良くする動きがありました。

アセットパイプラインとして「Sprockets」が利用されており、それが無い頃は特定のディレクトリに置き、それを読み込むことで画面を表示すると言うのが一般的でした。

しかし、アセットパイプラインが出たことで複数の JavaScript や CSS ファイルを圧縮・最小化・それらのファイル間の依存性を解決・ダイジェストの付加を可能としました。

JavaScript では「CoffeeScript」、CSS では「SCSS」が使われていましたが、アセットパイプラインによって純粋な JavaScript と CSS に変換されます。(これをコンパイルとよびます)

その後、Rails 5.1 からは、Webpack というモジュールバンドラーをサポートするようになりました。

Rails でサポートするため、Gem で Webpacker というライブラリを使うことで、Webpack が利用できるようになります。

Rails 6 系では、JavaScript は `Webpack`、CSS は `Sprockets` といった使い分けが標準設定となっていたようです。

Webpack の利点は、ES6 構文以降の新しい書き方を古いブラウザでも動くようにトランスパイルする Babel を使っていることです。

そして、Rails 7 系においては、Webpacker ではなく、新しく登場した Shakapacker が使われるようになるようです。

以上のように、 Rails はバージョンごとでクライアント側の開発事情が大きく変わっていっているようです。

## Webpackerの代替

Webpacker が使われなくなるということですが、どういった開発方法になるのでしょうか。

Webpacker に替わる方式として、新しい 4 つの Gem ライブラリが追加されています。

- propshaft
- importmap-rails
- jsbundling-rails
- cssbundling-rails

Webpacker を使わない代わり、Propshaft または Sprockets に、他のライブラリをアセット管理方式に合わせて組み合わせていく形になります。

元々存在していた Sprockets のアセットパイプラインに追加される形で今回 Propshaft が出てきました。

両者の違いとしては、トランスパイル、圧縮・連結などの処理をするかしないかになります。Propshaft は、トランスパイル、圧縮・連結などの処理をせず、ダイジェストの追加だけ実行するため、Sprockets よりも高速で動作するのが特徴です。


## importmaps-rails

公式 [Github](https://github.com/rails/importmap-rails) から引用しています。

>Import maps let you import JavaScript modules using logical names that map to versioned/digested files – directly from the browser. So you can build modern JavaScript applications using JavaScript libraries made for ES modules (ESM) without the need for transpiling or bundling. This frees you from needing Webpack, Yarn, npm, or any other part of the JavaScript toolchain. All you need is the asset pipeline that's already included in Rails.

以下日本語訳です。

>importmaps-rails は、バージョン管理されたファイルやダイジェストファイルに対応する論理名を使用し、ブラウザから直接 avaScript モジュールをインポートできます。
そのため、トランスパイルやバンドリングの必要なく、ES モジュール (ESM) 用に作られた JavaScript ライブラリを使用して、最新の JavaScript アプリケーションを構築できます。
これにより、Webpack、Yarn、npm、または JavaScript ツールチェインの他の部分が不要になります。必要なのは、Rails にすでに含まれているアセットパイプラインだけです。

簡単に説明すると下記のようなマッピングファイルがあるとします。

```js
<script type="importmap">
{
  "imports": {
    "moment": "/node_modules/moment/src/moment.js",
    "lodash": "/node_modules/lodash-es/lodash.js",
    "react": "https://ga.jspm.io/npm:react@17.0.2/index.js",
  }
}
</script>
```

上記のように記載しておけば、下記のように import を表現できるということです。

```js
import moment from "moment";
import { partition } from "lodash";
import React from "react";
```

サンプルの通り、node_modules からのインポート以外にもサイト URL も可能ですので、最新のパッケージを引っ張って使うことができるようになります。

### importmaps-railsの問題点

これがあれば Webpack とか使わなくなるので便利だなとおもったのですが、どうやら React ではあまり向いていないようです。

参考 URL：https://www.orzs.tech/importmap-rails-and-jsbuilding-rails/

読んでいただければわかりますが、React では基本 jsx(or tsx)で記載をしていき、jsx は JavaScript にトランスパイルをしてあげなければいけませんので厳しいですね。

ですので、React を使うのでしたら、「jsbundling-rails」の一択になりそうです。

## jsbundling-rails

jsbundling-rails は、Webpack や esbuild、rollup.js といったモジュールバンドラーの使用をサポートする Gem ライブラリです。

どのバンドラーを使うかについては、コマンドで指定して選ぶことが可能となります。

<!-- textlint-disable -->

近年 Webpacker の開発が中止されてしまったことで、Webpack を使いたい場合は、Webpacker を使うのではなく、jsbundling-rails の Webpack を使うのが良さそうです。

<!-- textlint-enable -->

## cssbundling-rails

cssbundling-rails は、Bootstrap や Dart Sass、PostCSS などをアプリケーションに組み込む際、必要に応じて使用します。


## rails newコマンド問題

これまで述べたとおり、クライアントサイド開発においては、さまざまな選択肢があります。

ですのでどういった構成で開発していくかを自由に決められる一方、どのような構成がいいか定まりにくいです。

個人的に知っておいた方が良い物をピックアップいたします。

- --database=DATABASE

標準が SQLite ですが、MySQL などに変更できます。そうすることで、config/database.yml がそれぞれのデータベースに適した記述となります。

- --JavaScript=JAVASCRIPT

JavaScript のモジュールバンドラーを選択できます。

選択肢としましては、importmap (default), webpack, esbuild, rollup となっています。

importmap がデフォルトですので、React を使う場合はこのオプションが必須となります。

- --force

強制上書きができますので、作り直しとかに便利です。

- --css=CSS

CSS プロセッサを選択できます。選択肢としましては tailwind・bootstrap・bulma・PostCSS・sass などがあるようです。

参考：https://github.com/rails/cssbundling-rails

## ローカルマシンでの開発環境構築

ここからは実際にローカル環境を構築する方法を記載します。

折角ですので、esbuild の Docker バージョンを構築していきます。

Webpack などでの方法は調べればたくさんあるのですが、esbuild となるとあまり文献がなく、手探りの方法ですのでもし他にいい方法があれば教えてください。

### Dockerの準備

Docker コンテナとしては、２つのコンテナを用意します。

- Web コンテナ(Rails Ver.7 アプリケーション稼働)
- DB コンテナ(MySQL Ver.8 サーバー)

開発バージョンは以下とします。

Ruby:3.1.1
Node:14.17.1
Rails:7.0.4

Web サーバーのコンテナもあれば良いですが、Web コンテナの中の Rails で development サーバーである puma が用意されているので、それを稼働して動かすようにします。

ディレクトリ構成は以下とします。

```
test_app
|-web
   |-.dockerignore
   |-Gemfile
   |-Gemfile.lock
   |-Dockerfile
   |-entrypoint.sh
|-docker-compose.yml
```

必要に応じて必要に応じて.node-version や.Ruby-version を置いてあげてください。

一旦下記の記載だけしておきます。


まずは、docker-compose.yml です。

```yaml
version: '3'
services:
  mysql:
    platform: linux/x86_64
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: password
    ports:
      - '3306:3306'
    command: --default-authentication-plugin=mysql_native_password
    volumes:
      - mysql-data:/var/lib/mysql
  web:
    build: ./web
    command: /bin/sh -c "rm -f tmp/pids/server.pid && ./bin/dev"
    volumes:
      - ./web:/myapp
      - rails_gem_data:/usr/local/bundle
    ports:
      - "3000:3000"
    depends_on:
      - mysql
    stdin_open: true
    tty: true

volumes:
  mysql-data:
  rails_gem_data:
    driver: local
```

web/Dockerfile

```Dockerfile
FROM --platform=amd64 ruby:3.1.1

RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
  && echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
  && apt-get update \
  && curl -fsSL https://deb.nodesource.com/setup_14.x | bash \
  && apt-get install -y nodejs cron \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* \
  && mkdir /myapp

RUN npm install --global yarn
RUN yarn install --network-timeout 600000

WORKDIR /myapp
COPY Gemfile /myapp/Gemfile

COPY Gemfile.lock /myapp/Gemfile.lock

RUN bundle install

COPY . /myapp

COPY entrypoint.sh /usr/bin/

RUN chmod +x /usr/bin/entrypoint.sh

ENTRYPOINT ["entrypoint.sh"]

EXPOSE 3000

CMD ["rails", "server", "-b", "0.0.0.0"]
```

Gemfile
```Gemfile
source "https://rubygems.org"
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

ruby "3.1.1"

# Bundle edge Rails instead: gem "rails", github: "rails/rails", branch: "main"
gem "rails", "~> 7.0.4"

# The modern asset pipeline for Rails [https://github.com/rails/propshaft]
gem "propshaft"
```

Gemfile.lock は一旦空ファイルで OK です。


entrypoint.sh

```sh
#!/bin/bash
set -e

# Remove a potentially pre-existing server.pid for Rails.
rm -f /myapp/tmp/pids/server.pid

# Then exec the container's main process (what's set as CMD in the Dockerfile).
exec "$@"
```


.dockerignore

```.dockerignore
tmp/
node_modules/
app/assets/builds/
public/assets/
log/
vendor/bundle/
coverage
.ideas/
data/
```


ここまで準備できたら一旦 image のビルドをしましょう。

```cil
$ docker-compose build --no-cache
```

ここまでまずできたら Docker の環境は問題ないです。

### Railsアプリの構築

ここからは、Rails アプリを入れていきます。


```cil
$ docker-compose run web rails new . --force --database=mysql -a propshaft -j esbuild --skip-hotwire
```

Rails new アプリのコマンド問題の通り、コマンド入力にたくさんのオプションが必要となります。

ここで、web コンテナで rails new コマンドを実行。

- database は MySQL(デフォが SQLite)
- アセットパイプラインは、propshaft
- モジュールバンドラーは、esbuild(デフォが importmap なので必須)
- Hotwire を組み込まない設定
- force で強制上書き


上記でコマンドを打つとエラーが起きます。

```
/usr/local/lib/ruby/3.1.0/bundler/rubygems_ext.rb:18:in `source': uninitialized constant Gem::Source (NameError)

      (defined?(@source) && @source) || Gem::Source::Installed.new
                                           ^^^^^^^^
Did you mean?  Gem::SourceList
	from /usr/local/lib/ruby/3.1.0/bundler/rubygems_ext.rb:50:in `extension_dir'
	from /usr/local/lib/ruby/3.1.0/rubygems/basic_specification.rb:339:in `have_file?'
	from /usr/local/lib/ruby/3.1.0/rubygems/basic_specification.rb:86:in `contains_requirable_file?'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:1038:in `block (2 levels) in find_in_unresolved_tree'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:2555:in `block (2 levels) in traverse'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:2550:in `each'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:2550:in `block in traverse'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:2548:in `each'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:2548:in `traverse'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:1034:in `block in find_in_unresolved_tree'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:1033:in `each'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:1033:in `find_in_unresolved_tree'
	from <internal:/usr/local/lib/ruby/3.1.0/rubygems/core_ext/kernel_require.rb>:114:in `require'
	from /usr/local/lib/ruby/3.1.0/bundler/rubygems_ext.rb:18:in `source'
	from /usr/local/lib/ruby/3.1.0/bundler/rubygems_ext.rb:50:in `extension_dir'
	from /usr/local/lib/ruby/3.1.0/rubygems/basic_specification.rb:339:in `have_file?'
	from /usr/local/lib/ruby/3.1.0/rubygems/basic_specification.rb:86:in `contains_requirable_file?'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:1038:in `block (2 levels) in find_in_unresolved_tree'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:2555:in `block (2 levels) in traverse'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:2550:in `each'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:2550:in `block in traverse'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:2548:in `each'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:2548:in `traverse'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:1034:in `block in find_in_unresolved_tree'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:1033:in `each'
	from /usr/local/lib/ruby/3.1.0/rubygems/specification.rb:1033:in `find_in_unresolved_tree'
	from <internal:/usr/local/lib/ruby/3.1.0/rubygems/core_ext/kernel_require.rb>:114:in `require'
	from /usr/local/lib/ruby/3.1.0/bundler/vendor/net-http-persistent/lib/net/http/persistent.rb:1:in `<top (required)>'
	from /usr/local/lib/ruby/3.1.0/bundler/vendored_persistent.rb:11:in `require_relative'
	from /usr/local/lib/ruby/3.1.0/bundler/vendored_persistent.rb:11:in `<top (required)>'
	from /usr/local/lib/ruby/3.1.0/bundler/fetcher.rb:3:in `require_relative'
	from /usr/local/lib/ruby/3.1.0/bundler/fetcher.rb:3:in `<top (required)>'
	from <internal:/usr/local/lib/ruby/3.1.0/rubygems/core_ext/kernel_require.rb>:85:in `require'
	from <internal:/usr/local/lib/ruby/3.1.0/rubygems/core_ext/kernel_require.rb>:85:in `require'
	from /usr/local/lib/ruby/3.1.0/bundler/cli/install.rb:50:in `run'
	from /usr/local/lib/ruby/3.1.0/bundler/cli.rb:255:in `block in install'
	from /usr/local/lib/ruby/3.1.0/bundler/settings.rb:131:in `temporary'
	from /usr/local/lib/ruby/3.1.0/bundler/cli.rb:254:in `install'
	from /usr/local/lib/ruby/3.1.0/bundler/vendor/thor/lib/thor/command.rb:27:in `run'
	from /usr/local/lib/ruby/3.1.0/bundler/vendor/thor/lib/thor/invocation.rb:127:in `invoke_command'
	from /usr/local/lib/ruby/3.1.0/bundler/vendor/thor/lib/thor.rb:392:in `dispatch'
	from /usr/local/lib/ruby/3.1.0/bundler/cli.rb:31:in `dispatch'
	from /usr/local/lib/ruby/3.1.0/bundler/vendor/thor/lib/thor/base.rb:485:in `start'
	from /usr/local/lib/ruby/3.1.0/bundler/cli.rb:25:in `start'
	from /usr/local/lib/ruby/gems/3.1.0/gems/bundler-2.3.7/libexec/bundle:48:in `block in <main>'
	from /usr/local/lib/ruby/3.1.0/bundler/friendly_errors.rb:103:in `with_friendly_errors'
	from /usr/local/lib/ruby/gems/3.1.0/gems/bundler-2.3.7/libexec/bundle:36:in `<main>'
         run  bundle binstubs bundler
Could not find gem 'propshaft' in locally installed gems.
       rails  javascript:install:esbuild
Could not find gem 'propshaft' in locally installed gems.
Run `bundle install` to install missing gems.
```

propshaft が見つからない...。

```
Could not find gem 'propshaft' in locally installed gems.
```


bundle install すればいいと書いてあるが、少し気持ち悪い..。

とりあえず、言われた通りに実行します。

```cil
$ docker-compose run --rm web bundle install

.
.
.
Installing bootsnap 1.15.0 with native extensions
Bundle complete! 13 Gemfile dependencies, 69 gems now installed.
Use `bundle info [gemname]` to see where a bundled gem is installed.
Post-install message from rubyzip:
RubyZip 3.0 is coming!
**********************

The public API of some Rubyzip classes has been modernized to use named
parameters for optional arguments. Please check your usage of the
following classes:
  * `Zip::File`
  * `Zip::Entry`
  * `Zip::InputStream`
  * `Zip::OutputStream`

Please ensure that your Gemfiles and .gemspecs are suitably restrictive
to avoid an unexpected breakage when 3.0 is released (e.g. ~> 2.3.0).
See https://github.com/rubyzip/rubyzip for details. The Changelog also
lists other enhancements and bugfixes that have been implemented since
version 2.3.0.
```

今度は成功しましたので、無事に Rails が入りました。

しかし、これでは、propshaft が入っていませんので、`rails  javascript:install:esbuild`が実行できていません。個別に対応していきます。

Gemfile には、`jsbundling-rails`が入っていますので、[2番](https://github.com/rails/jsbundling-rails)のコマンドを実行します。

```cil
$ docker-compose run --rm web rails  javascript:install:esbuild

[+] Running 1/0
 ⠿ Container test_app-mysql-1  Running                                                                                                          0.0s
Compile into app/assets/builds
      create  app/assets/builds
      create  app/assets/builds/.keep
      append  .gitignore
      append  .gitignore
Add JavaScript include tag in application layout
      insert  app/views/layouts/application.html.erb
Create default entrypoint in app/javascript/application.js
      create  app/javascript
      create  app/javascript/application.js
Add default package.json
      create  package.json
Add default Procfile.dev
      create  Procfile.dev
Ensure foreman is installed
         run  gem install foreman from "."
Fetching foreman-0.87.2.gem
Successfully installed foreman-0.87.2
1 gem installed
Add bin/dev to start foreman
      create  bin/dev
Install esbuild
         run  yarn add esbuild from "."
yarn add v1.22.19
info No lockfile found.
[1/4] Resolving packages...
[2/4] Fetching packages...
[3/4] Linking dependencies...
[4/4] Building fresh packages...
success Saved lockfile.
success Saved 2 new dependencies.
info Direct dependencies
└─ esbuild@0.16.8
info All dependencies
├─ @esbuild/linux-x64@0.16.8
└─ esbuild@0.16.8
Done in 23.46s.
Add build script
Add "scripts": { "build": "esbuild app/javascript/*.* --bundle --sourcemap --outdir=app/assets/builds --public-path=assets" } to your package.json
```

上記コマンドで esbuild に必要なファイルや yarn によるパッケージインストールが行われます。

最後の指示で、package.json にコマンドを追加しろと書いてあるので、追加していきます。

```json
{
  "name": "app",
  "private": "true",
  "dependencies": {
    "esbuild": "^0.16.8"
  },
  // 以下追記
  "scripts": {
    "build": "esbuild app/javascript/*.* --bundle --sourcemap --outdir=app/assets/builds --public-path=assets"
  }
}
```

これまでは、puma サーバー起動に関しては、`bin/rails server`で起動していましたが、`bin/dev`と`Procfile.dev`ファイルが生成しています。

つまり、foreman を経由でサーバーを起動するように変更されています。

foreman については説明を割愛しますが、Heroku でもこの機能は使われています。

`docker-compose.yml`に戻りますが、それを見越し、web コンテナのコマンドをこのようにしていました。

```
command: /bin/sh -c "rm -f tmp/pids/server.pid && ./bin/dev"
```

foreman 経由でのサーバー起動コマンドであらかじめ準備しておきました。

Procfile.dev の web コマンドは少し編集します。下記を参考ください。

```Procfile.dev
web: bin/rails server -p 3000 -b '0.0.0.0'
js: yarn build --watch
```


### Databaseの設定変更

database を MySQL にしているので、config/database.yml のファイルを編集します。

```yml
default: &default
  adapter: mysql2
  encoding: utf8 #utf8mb4から変更
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: root
  password: password #docker-compose.ymlの定義に変更
  host: mysql #mysqlコンテナ名に変更
```

データベースを作ってみましょう。

```cil
$ docker-compose run --rm web rails db:create

[+] Running 1/0
 ⠿ Container test_app-mysql-1  Running                                                                                                          0.0s
Created database 'myapp_development'
Created database 'myapp_test'
```

成功しました。ここまでできると画面表示の準備は完了です。

念の為ビルドしておきましょう。

```cil
$ docker-compose build --no-cache
```


### 画面表示

コンテナを起動しましょう。

```cil
$ docker-compose up
.
.
est_app-web-1    | 03:31:25 web.1  | => Booting Puma
test_app-web-1    | 03:31:25 web.1  | => Rails 7.0.4 application starting in development
test_app-web-1    | 03:31:25 web.1  | => Run `bin/rails server --help` for more startup options
test_app-web-1    | 03:31:27 web.1  | Puma starting in single mode...
test_app-web-1    | 03:31:27 web.1  | * Puma version: 5.6.5 (ruby 3.1.1-p18) ("Birdie's Version")
test_app-web-1    | 03:31:27 web.1  | *  Min threads: 5
test_app-web-1    | 03:31:27 web.1  | *  Max threads: 5
test_app-web-1    | 03:31:27 web.1  | *  Environment: development
test_app-web-1    | 03:31:27 web.1  | *          PID: 31
test_app-web-1    | 03:31:27 web.1  | * Listening on http://0.0.0.0:3000
test_app-web-1    | 03:31:27 web.1  | Use Ctrl-C to stop
```


無事に画像表示されたら OK です。

![rails-top](/images/rails-top.png)


## esbuildによるJavaScript編集方法(React導入)

これで esbuild を使う環境は整いました。

では、ここからはどうやって JavaScript を編集&React を入れるのかを説明していきます。

app/JavaScript の中に`application.js`があるはずです。

ここが、ビルドされる JS ファイルのエントリーポイントとなるファイルです。

今、js ファイルとなっていますが、これに React を入れていきます。

```cil
$ docker-compose run --rm web yarn add react react-dom
```

入れた後に、Rails の MVC モデルの部分を作成していきますが、Rails 側の MVC は１つだけあれば十分ですので、下記で１セットだけ準備します。


```cil
$ docker-compose run --rm web rails g controller home index

[+] Running 1/0
 ⠿ Container test_app-mysql-1  Running                                                                                                          0.0s
      create  app/controllers/home_controller.rb
       route  get 'home/index'
      invoke  erb
      create    app/views/home
      create    app/views/home/index.html.erb
      invoke  test_unit
      create    test/controllers/home_controller_test.rb
      invoke  helper
      create    app/helpers/home_helper.rb
      invoke    test_unit
```

config/routes.rb を編集します。

```rb
Rails.application.routes.draw do
  root 'home#index'
  # Define your application routes per the DSL in https://guides.rubyonrails.org/routing.html

  # Defines the root path route ("/")
  # root "articles#index"
end
```

そうするとルート(/)では、home コントローラーの index メソッドが呼ばれます。view は、`app/views/home/index.html.erb`が描画されるので、画面が変わるはずです。

そして、お次に大元の Layouts である、`application.html.erb`を編集します。

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Myapp</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>

    <%= stylesheet_link_tag "application" %>
    <%= javascript_include_tag "application", "data-turbo-track": "reload", defer: true %>
  </head>

  <body>
    <div id="root"></div> ←追加
    <%= yield %>
  </body>
</html>
```

`<div id="root"></div>`を追加することで、React の JSX による描画を表示します。

最後に、`application.js`を`application.jsx`に変更し、React の JSX 記法で記述していきます。

```jsx
import React from 'react';
import { createRoot } from 'react-dom/client';

function App() {
  return (
    <h1>Hello React!</h1>
  )
}

const root = document.getElementById('root');
if (!root) {
  throw new Error('No root element');
}
createRoot(root).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

React の描画だけにしたいので、`app/views/home/index.html.erb`の中身は空にしましょう。

こうすると、このように React で描画されて表示されているのがわかります。

![react-top](/images/react-rendering.png)

`application.jsx`がエンドポイントファイルとなるので、ここを基準にファイルを作っていくと良いです。


## まとめ

いかがでしたでしょうか。

Rails の中に React を組み込んでしまったらとてもスッキリし開発もやりやすくなるはずです。

今までは、Webpack しかあまり知識を得ておらず、esbuild についての知見が浅く、ここまで作るのに苦労しました。

よければ皆さんも触ってみてください。

あとは React を TypeScript にしたり、ESLint や Prettier を入れたりとお好きな環境にカスタマイズしてみてください。

今度は GraphQl による通信方法を記事にします。
