---
title: "Docker環境にメールサーバー構築でMailhogを利用する"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Laravel", "Docker", "Mailhog"]
published: true
---

# Docker にメールサーバ導入

Laravel を nginx と MySQL で構築した Docker 環境にメールサーバを追加で構築しました。

結構簡単に導入できるので、手順としてまとめています。

# Mailhog とは

MailHog は`Go言語`で書かれているメールサーバで、`letter_opener_web`と同様にブラウザでメールの内容を確認できるようになります。

また、MailHog 単体でメールサーバーとして立ち上げることができるため、Laravel や Rails に限らずどんな環境でも利用できます。

[mailhog/MailHog](https://github.com/mailhog/MailHog)

ちなみに、Laravel の最新バージョン 8 の sail のパッケージで、使用されています。

その他サービスとして`Maildev`や`MailCatcher`があるが、それらは`gem`や`npm`でインストールしてからじゃないと使えないのがとても面倒...。

今回、言語によって使えるサービスが異なるよりも、どんな環境でも汎用的に利用できる Mailhog を利用することとした。

# 環境構築手順

## Docker-compose.yml を編集

下記を`docker-compose.yml`に追記します。

```yaml
#↓下記追記する↓
#メールサーバのコンテナ
mail:
  image: mailhog/mailhog
  container_name: mailhog
  ports:
    - "8025:8025"
  environment:
    MH_STORAGE: maildir
    MH_MAILDIR_PATH: /tmp
```

実際の SMTP は、port：1025 で受け付けている。

### Volume の設定

mailhog はメールをメモリ上に保存するため、Docker のコンテナを停止するとメールが消えてしまう。

そのため、volume の設定は必須のため、`volumes`に`maildir: {}`、mail コンテナ部 `volumes: - maildir:/tmp`を記載します。

```yaml
volumes:
  db-volume:
  maildir: {}　#⇦追記する

  mail:
    image: mailhog/mailhog
    container_name: mailhog
    ports:
      - '8025:8025'
    environment:
      MH_STORAGE: maildir
      MH_MAILDIR_PATH: /tmp
　　#↓追記する
    volumes:
      - maildir:/tmp
```

### Docker-compose.yml の完成形はこちら

```yaml
#docker-composeバージョン
version: "3.8"

volumes:
  db-volume:
  maildir: {}

#コンテナ詳細
services:
  #Webサーバーのコンテナ
  nginx_server:
    image: nginx:1.18
    container_name: nginx-server
    ports:
      - "8000:80"
    #コンテナの依存関係を示す(PHP→Nginxの順)
    depends_on:
      - php
    volumes:
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
      - ./:/var/www/html/

  #phpアプリケーションのコンテナ
  php:
    build:
      context: ./docker/php
      dockerfile: Dockerfile
    container_name: php-app
    ports:
      - "9000:9000"
    volumes:
      - ./:/var/www/html/

  #データベースのコンテナ
  mysql:
    image: mysql:8.0
    container_name: mysql
    ports:
      - "4306:3306"
    environment:
      MYSQL_DATABASE: crm_test_db
      MYSQL_USER: admin
      MYSQL_PASSWORD: CrmTest2021
      MYSQL_ROOT_PASSWORD: root
      TZ: "Asia/Tokyo"
    volumes:
      - db-volume:/var/lib/mysql

  #メールサーバのコンテナ
  mail:
    image: mailhog/mailhog
    container_name: mailhog
    ports:
      - "8025:8025"
    environment:
      MH_STORAGE: maildir
      MH_MAILDIR_PATH: /tmp
    volumes:
      - maildir:/tmp
```

## Docker の再ビルド

mailhog のイメージを新たに使用するため、image のビルドが必要。

下記コマンドを実行し、イメージのビルドとコンテナ起動をします。

```yaml
# イメージのビルド
$ docker-compose build

# コンテナ起動
$ docker-compose up -d
```

## メールサーバ起動確認

### コンテナ確認

下記コマンドにてメールサーバ(Mailhog)が起動していることを確認する。

```jsx
$ docker-compose ps
        Name                       Command              State                 Ports
-------------------------------------------------------------------------------------------------
docker-laravel_app_1    docker-php-entrypoint php-fpm   Up      9000/tcp
docker-laravel_db_1     docker-entrypoint.sh mysqld     Up      0.0.0.0:3306->3306/tcp, 33060/tcp
docker-laravel_mail_1   MailHog                         Up      1025/tcp, 0.0.0.0:8025->8025/tcp
docker-laravel_web_1    nginx -g daemon off;            Up      0.0.0.0:80->80/tcp
```

### 画面確認

http://localhost:8025 にアクセスし、Mailhog 画面が見られることを確認する。

## Laravel 側の設定

ここからは、`phpコンテナ`にインストールしている`Laravelプロジェクト`の設定になります。

### Laravel の.env を修正

下記.env の通り修正。

`MAIL_FROM_ADDRESS`は好きなアドレスで構いません。

```jsx
MAIL_HOST=mail
MAIL_PORT=1025
MAIL_FROM_ADDRESS=info@example.com
```

### メール送信テスト(簡易テスト)

tinker でメールを送って送信テストを実施する。

response が`null`なら成功です。

```java
// 下記コマンドでphpコンテナに入り、php atrtisan tinkerを実行できます
$ docker-compose exec php php artisan tinker
>>> Mail::raw('test mail',function($message){$message->to('test@example.com')->subject('test');});

=> null
>>>
```

Mail::raw('test mail',function($message){$message->to('[mattu.nao722@gmail.com](mailto:mattu.nao722@gmail.com)')->subject('test');});

## Mailhog 確認

http://localhost:8025
にアクセスし、Mailhog 画面でメールが受信されていることを確認する。

今回しっかり送信され、test mail というメッセが入っていることも確認した。

## ルーティングを絡めた送信確認(Mail ファサード使用)

あくまで tinker を使うと簡易的確認であるため、ここからはルーティングとコントローラーを絡めた送信テストを実施する。

### コントローラー作成

メール送信テストのためのコントローラーを作成します。

```yaml
$ php artisan make:controller MailSendController
```

### ルーティング設定

routes/web.php にルーティング設定を実施し、CRUD を実装。

(今回は get メソッドでメール送信を実行する)

```php
# 追記
use App\Http\Controllers\MailSendController;

# 追記
Route::get('/mail', [MailSendController::class, 'index']);
```

### コントローラー編集

```php
namespace App\Http\Controllers;

use Illuminate\Http\Request;

use Mail;

class MailSendController extends Controller
{
    public function index(){

    	$data = [];

    	Mail::send('emails.test', $data, function($message){
    	    $message->to('abc987@example.com', 'Test')
    	    ->subject('This is a test mail');
    	});
    }

}
```

- `to`には送信した宛先のメールを入力
- `subject`にはメールのタイトルを設定
- メールの中身は`emails.test`に記述

### test.blade.php(メール内容)編集

emails.test はブレードファイルを示している。

そのため、resources ディレクトリの下に emails ディレクトリを作成し、test.blade.php を作成します。

welcome.blade.php の中身は下記を記述します。

(メール内容に関わるだけで適当でいいですが、今回メールリンクも入れてみました)

```html
<!DOCTYPE html>
<html lang="ja">
  <style>
    body {
      background-color: #fffacd;
    }
    h1 {
      font-size: 16px;
      color: #ff6666;
    }
    #button {
      width: 200px;
      text-align: center;
    }
    #button a {
      padding: 10px 20px;
      display: block;
      border: 1px solid #2a88bd;
      background-color: #ffffff;
      color: #2a88bd;
      text-decoration: none;
      box-shadow: 2px 2px 3px #f5deb3;
    }
    #button a:hover {
      background-color: #2a88bd;
      color: #ffffff;
    }
  </style>
  <body>
    <h1>Sample Notification!</h1>
    <p>A sample notification has been sent.</p>
    <p id="button">
      <a href="https://www.google.co.jp">リンクのテスト</a>
    </p>
  </body>
</html>
```

### メール送信テスト

`https://localhost/8000/mail`へアクセス。

アクセス後、画面は表示しないが、Mailhog を見るとメール送信されているのが確認できる。

# 参考記事

[Laravel で MailHog を導入して快適にメール開発をしよう！【Docker】](https://tech.windii.jp/backend/laravel/laravel-mailhog-docker-compose)

[Docker × Laravel メールの送信処理をローカルで確認する](https://laptrinhx.com/docker-laravel-meruno-song-xin-chu-liworokarude-que-rensuru-1410764577/)

[開発環境で役に立つメール送信テストツールたち](https://www.metrocode.co/blog/post/mail-testing-tools)
