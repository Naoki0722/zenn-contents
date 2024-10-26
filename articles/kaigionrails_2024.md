---
title: "Kaigi on Rails 2024 参加レポート"
emoji: "🗒️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
publication_name: "yamaden"
---

初めて Kaigi on Rails に参加しました。

そのレポート的なものをこちらにまとめます。

※私的メモかつ個人の主観が混じっておりますのでその点ご了承ください。


今回聞きたいことの目玉としては、以下が気になっていました。

- Hotwire 関連の話
- 今は使わなくなった GraphQL について
- ViewComponent
- The One Person Framework の考え方


# 1日目(10/25)
## 推し活としてのrails new

https://kaigionrails.org/2024/talks/sakahukamaki/
https://speakerdeck.com/sakahukamaki/oshikatsu-ha-iizo

### 概要

- マシュマロは rails と Python を使ってるそう
  - 機械学習で Python をつかっていそう
- できないことにこだわるとぽしゃる(こだわったデザインとか)
  - まずはリリースして使われることを意識する
  - 2023/6/5 に開発スタートして 10 日にはリリースしたので実質開発期間は 5 日
- Rails は何も考えずに作れるのがいい
- 稼働環境としては、いろいろなサーバーがある
- 国産で Rails を動かしたかったということもあり以下を採用
  - ロリポップマネージドクラウド
- 監視とバックアップはやっていない

### 所感

本当に最低限の機能と構成で実際にデプロイして運用している経験を聞かせてもらいました。

スライドの中にもあったように、以下の点を自分の個人開発でも意識してみるとうまくいきそうです。

- ゆるくやる
- 息切れしないで運用する
- 終わらせないのが大事

どうしても周囲の同じぐらいの経験年数の方々と比較しがちになり、今の環境がどうしても自分の技術的な成長意欲に対してあっていない気がしていました。

この辺りの考え方は、今後自分がこの業界で長くやっていくコツなのかも知れないと思いました。

また、何か使えるものを最小限で作るという考え方は今の業務にも活かせる部分があります。

必要最低限の機能だけを実装する視点で、不要な機能はどんどん減らし、システムのボリュームはなるべく軽くしたいなと思いました。

## Sidekiqで実現する長時間の非同期処理の中断と再開

https://kaigionrails.org/2024/talks/hypermkt/
https://speakerdeck.com/hypermkt/pausing-and-resuming-long-running-asynchronous-jobs-with-sidekiq

### 概要

- 非同期処理の中断と再開について(Sidekiq)Redis を利用している
- Sidekiq には無償と有償版がある
  - ちなみにうちでは現在 Sidekiq の代わりに「Resque」を利用している
  - https://github.com/resque/resque
- 非同期処理(JOB)の問題点
  - 途中で止めることができない
    - データ不整合や再度実行による処理時間の増加
- 回避策として
  - 長時間実行されていない JOB がないこと、途中で止めても影響ないことを確認してデプロイ

上記を改善するために中断・再開機能を実装したとのことです。

中断・再開のメリットは以下の通りです。

- 安心安全に非同期処理を止め、デプロイがしやすくなる。
- 途中で JOB を止めなくても良くなるため、JOB の実行時間短縮が図れる

中断処理の対応として以下のようなものでした。

- 終了 or 停止を検知するフラグを initializer に設定
- Sidekiq の処理終了を検知したらフラグを切り替え
- フラグを利用して中断するかの判定材料として利用する
- 中断処理を入れるタイミングのポイント
  - 繰り返しの最後で組み込む
- 再開処理については２種類
  - ID 番号 保持方式
    - JOB のユニークキーを持たせてその ID を利用して再開処理する
  - 行番号 保持方式
    - 主に CSV ファイル系など

- Sidekiq のライブラリに再開中断を設定できる機能が ver 7.3 にリリースされたらしい
  - https://github.com/sidekiq/sidekiq/blob/main/Changes.md#730
  - https://github.com/sidekiq/sidekiq/wiki/Iteration

### 所感

うちでも、Resque を使った JOB 機能を利用しており、cron によって定期実行されています。

また、注文完了時のデータ不整合の問題もありつつ、デプロイが日中にできない等の問題を抱えておりました。
注文完了時のデータ不整合の対応としてユーザーには操作させないよう、メンテナンスモードを実装することで防ぐことはできるようになりました。

しかしながら、JOB 実行中を避けてデプロイしなければならないという課題は残ったままです。

今後基幹開発が進むことでどの時間に JOB が実行されているのでこの時間の deploy は避けないといけないということが発生しうることとなります。

また、印刷関係も JOB で作ることで、日中の社内業務に影響を与えてしまうため、不具合が起きた時の荷中 deploy をできるように考えていかなければならないと思いました。

## カスタムしながら理解するGraphQL Connection

https://kaigionrails.org/2024/talks/yana-gi/
https://speakerdeck.com/yanagii/kasutamusinagarali-jie-surugraphql-connection

### 概要

- minne は Next+Rails の構成になってる
- ページネーションの種類
  - オフセットページネーション
  - カーソルページネーション
- Custom Connection とは
  - https://graphql-ruby.org/pagination/custom_connections
  - 配列データの取得時にページネーションをする仕組みのこと
  - Graphql::Pagiination::Connection クラスを継承してカスタムするよう
    - https://github.com/rmosolgo/graphql-ruby/blob/master/lib/graphql/pagination/connection.rb

### 所感

この辺については正直意識せず GraphQL を扱っていたので、今でも使い続ける選択をしていたらもう少し勉強してみたいと思いました。

またページネーションの種類についてカーソルとオフセットのに種類があり、この辺についてちゃんと理解していなかったので時間ある時に確認しておきたいです。

https://note.com/tomo_program/n/nbb010ff6eede
https://speakerdeck.com/nearme_tech/cursor-paginationnituite


## リリース8年目のサービスの1800個のERBファイルをViewComponentに移行した方法とその結果

https://kaigionrails.org/2024/talks/katty0324/
https://speakerdeck.com/katty0324/ririsu8nian-mu-nosabisuno1800ge-noerbhuairuwoviewcomponentniyi-xing-sitafang-fa-tosonojie-guo

### 概要

- 1800 ほどの View レイヤーが抱えていた問題
  - パラメータ定義の一貫性の無さ
    - コンポーネント(共通化)しにくい
    - 変数の把握が難しい
    - ローカル変数とインスタンス変数がごちゃごちゃ
  - erb 内のロジックの紛れ込み
    - 保守性・可読性が落ちる
    - ロジックコードでエラー発生の可能性がある
      - エラーの改善のため、テストを書きたいが view ファイルのテストは E2E でしかカバーできない
- 解決策として ViewComponent
  - https://viewcomponent.org/
  - 明確なインタフェース提供
  - テストも書ける
  - View ごとにクラスとそれに紐づく view があるイメージ
  - その中のクラスでロジックを記述できる
    - クラスファイルで記述
  - テストが書きやすい
    - コンポーネントの view デザインのテストも可能であれば、その中のロジックもテストできる
  - コードジャンプしやすい
- 大量の erb ファイルをどう ViewComponent に置き換えるか
  - 自動生成スクリプトを自作し、置き換え対応
- 置き換えの段階的移行
  - view ディレクトリのどちらを使うかを切り替えることができるようにする
  - カナリアリリース
    - 一部のユーザーにのみリリースすることを指す
    - 今回は 10％の確率で view2 側の view を読み込むように対応

### 所感

基幹開発を進める際、UI は割と複雑な表示をする可能性があります。

現在導入はしていないものの、導入検討してもいいかなと思いました。

初めて知ったこととしては、`ApplicationController.view_paths` で Rails が参照している view のパス が確認できるのは恥ずかしながら知りませんでした。

## ActionCableなら簡単 生成 AIの応答をタイピングアニメーションで表示。実装、コスト削減、テスト、運用まで。

https://kaigionrails.org/2024/talks/kaibadash/
https://docs.google.com/presentation/d/1sPCFlWPKmnTcc11Nt99swIJ-M056JtdK2tqHYZ18QEs/edit#slide=id.g300fd7ef164_0_33

### 概要

- LLM の API の応答をタイピングアニメーションで表示し UX を向上したい
  - タイミングアニメーションとは
  - GPT のようなリアルタイムで人が入力しているかのようなアニメーション
- 単純に API として組み込むと AI のレスポンスを全て合体して全て返す
- これを ActionCable を使い、双方向接続をすることでリアルタイムにレスポンスを返すという実装をした
  - コネクションを貼り続ける問題、既存の API サーバーに負荷をかける問題等がありそう
    - Rails ガイドでは ActionCable 専用サーバーを使ったほうがいいと書いてあった...
    - ちゃんとドキュメントは読もう...
- 重たいプロンプトが来たらどうするの
  - 非同期処理の JOB に任せる
  - JOB(Worker)から ActionCable って呼べるの
    - サブスクリプションアダプタというのがあるため可能
    - そうすると Redis が ActionCable に返せる
    - ちゃんとドキュメントよもう...
  - Redis の」quee が順番保証していない
    - index をつけて順番保証

- LLM のコスト削減
  - テストコードを書いて VCR を使う案
    - VCR は、1 回だけ本物のリクエストを送って記録してそれを使う方法
    - https://github.com/vcr/vcr
  - ローカルで LLM を動かす
    - LM Studio を使う
    - https://lmstudio.ai/
    - リソース食うらしい...
    - サーバー起動可能

### 所感

Lock の開発のため ActionCable は導入したばかりでとても勉強になりました。

今の構成がどうなっているかわかりませんが、今後基幹で画面アクセスが増えることで ActionCable のコネクションは増えるはずです。
ActionCable 専用のサーバーを持つべきかは相談したいです。

また、手軽に Local で LLM を扱えるライブラリやツールを知ることができたので、何か今後それ関連の個人開発等をする時があれば使ってみたいです。


## 現実のRuby/Railsアップグレード

https://kaigionrails.org/2024/talks/takeyuweb/
https://zenn.dev/takeyuwebinc/articles/4c44a4f284d3d1

### 概要
- 段階的アップデート
- Rails
  - bundle update rails
  - bundle exec rails app:update
    - 基本上書き
  - テスト実行
  - 運用に回して様子見
- わからない設定は、最終的には rails のソースコードを読むようにする
- 保守されていない Gemfile
  - Fork して自分で保守
  - 別 Gem に置き換える等して基本使わないようにする
- Rails をモンキーパッチしている Gem
  - これも基本使わない...
- 古い仕様に対する互換性はいずれ壊れる...
  - 非推奨はなるべく解決する
- rails7.2 から YJIT でパフォーマンスが改善された
- 余裕があったらやるはやらない
  - 直接利益を産まないからコストかけたくない
  - リスクを伝え、しっかりリソースを確保する

### 所感

フレームワークのアップデートなどは実際の企業活動には直接影響せず、否定的なイメージを持ちがちです。

ですが、ちゃんとエンジニア側からアップデートをしないことによるリスクや懸念をつたえ、積極的にやっていくべきだなと思いました。


## Hotwire or React 〜Reactの録画機能をHotwireに置き換えて得られた知見〜

https://kaigionrails.org/2024/talks/haruna-tsujita/
https://speakerdeck.com/harunatsujita/hotwire-or-react


### 概要
- React で実装された録画機能を Hotwire に置き換える
- 実際に置き換えた結果
  - Stimilus を使うことで JS コードが増える
  - ロジックが増えてしまってたから
  - プロポーザル提出時はやはり React の方が良いのでは という結論を出しそうになってた
- Turbo の使い心地は良かった
  - もしかしたら Turbo をちゃんと使いこなせていない
  - こっちに実装を寄せるべきでは
- この考えで実装
  - ユーザー操作動きで考えるのが良くなかった
  - 紙芝居そのもの→Turbo
  - 紙芝居の仕掛け→Stimulus
  - 結果として Stimilus がシンプルになった
  - Hotwire の規律というものを理解するべきだった
- Hotwire と React の選択
  - CRUD が絡むと Hotwire がいい
  - CRUD なしのインタラクティブなものは React のほうがいい
  - Hotwire 的な設計を使えばある程度のものは実装可能

### 所感

現在 Hotwire を使って実装しなおしており、今後開発が進むにつれ、Stimilus を使う場面はどんどん増えていくはずです。

その場合に Stimilus の実装が複雑になっていくことも考えられるため、Hotwire 的設計から Turbo で実装できないかを考えることがとても重要だと気づけました。

紙芝居的な考えを持って実装していけたらと思いました。

# 2日目(10/26)


## 作って理解する RDBMSのしくみ
https://kaigionrails.org/2024/talks/ydah/
https://speakerdeck.com/ydah/zuo-tuteli-jie-suru-rdbmsnosikumi

### 概要

- 6 月 kansai rubykaigi ある
- ANDPAD エンジニア
  - クラウド型建設プロジェクト管理サービス
- zig で作ってるそう
- RDBMS の仕組みについて
  - 構文解析器
    - クエリを解析して AST に変換
    - 字句解析
      - 意味ある単位でトークン変換
    - 構文解析
  - プランナ/オプティマイザ
    - AST を元に最適なクエリの実行計画を立てる
    - EXPLAIN 分で確認可能なもの
  - クエリエクスキューター
    - 実行計画通りにデータアクセス
  - バッファプールマネージャ
    - ディクスアクセスを減らす役割
    - LRU
      - 長い時間アクセスされていない要素を削除する戦略
  - ディクスマネージャー
      - 物理的な読み書きを管理
- インデックスがないとどうなる
  - 格納されたデータに順次アクセス
  - あるとルールごとに並び替えが可能になる
  - 種類
    - B-tree
    - Hash
    - GIN
    - Gist/SP-Gist

好きな言葉で以下のようなことをおっしゃってました。

- 経験を得ると 1 回目と 2 回目の景色は違う

### 所感

今ある機能をそのまま使いこなすことも大事ですが、その機能がどのように動いているのかを理解することも大切だと思いました。

少し時間があればいろんなことに興味を持って調べるということをやってみたいなと思いました。(例：Rails の ActiveRecord クラスがどのような動きをしているのか等)


## Cache to Your Advantage: フラグメントキャッシュの基本と応用

https://kaigionrails.org/2024/talks/tkawa/
https://drive.google.com/file/d/1Y3saUdJ98bLziDe94-4KTdughGqfAoHP/view

### 概要

- フラグメントキャッシュ
  - 検索や API 取得の処理に時間がかかると困るため、キャッシュを設置
  - 断片的なキャッシュ
  - キャッシュしたい部分を cache @product do のようにインスタンスを渡す
  - キャッシュはキーとバリューの組み合わせ
    - @product がキーとなる
    - 価格が変わったら変えないといけない
    - 古いキャッシュになるため消さないといけない、こういう問題が起きる
- 古いキャッシュを消すのは
  - DHH は推奨していない
  - キーベースのキャッシュ失効(消しているわけではない)
    - [モデル]/[id]-[updated_at]
    - updated_at が更新されるので消さなくていい
  - 複数モデルの時は (ActiveRecord Relation)
    - [モデル名]/query-[sql のハッシュ値]-[個数]-[updated_at]
  - has_many モデルの時
    - belongs_to touch: true というオプションが親を更新できる
- view もキャッシュされる
- 全てお任せでキャッシュしてくれるということ

全体ページを以下に分けたら良いと話されてました。

- 静的コンテンツ
- 動的コンテンツ
- キャッシュ NG
  - TurboFrame で解決
  - 以外と少ない
- ページ全体に適用した方が早いのでは
  - フラグメントキャッシュをページ全体に広げる
- セッションや Cookie をキャッシュに使ってはいけない
  - 自動的に セッション、Cookie が無効

### 所感

うちのシステムの場合、ログインユーザーごとに価格表示を変えたりすることが多いです。

また、毎時の在庫情報更新によって updated_at が変わってしまうため、キャッシュするメリットは無さそうに感じました。

ただ、管理画面側のデータ表示においては滅多に変更されないデータ件数の多いものはキャッシュして表示速度を上げる改善をしてもいいかなと思ってます。

https://railsguides.jp/caching_with_rails.html


## OmniAuthから学ぶOAuth 2.0

https://kaigionrails.org/2024/talks/ykpythemind/
https://speakerdeck.com/ykpythemind/omniauthkaraxue-buoauth2-dot-0-kaigi-on-rails-2024

### 概要

- OmniAuth とは
  - 認証を標準化するアプリ
- 認証、認可とは
  - 認証：個人を識別
  - 認可：対象物に権限付与
- Rails で
  - 認証実装
    - authenticate_by というメソッド
      - has_secure_password にある
  - 認可
    - ロール
    - before_action で管理者かどうかをチェックしてダメなら forbiidden
- Google でログインさせたい(OmniAuth)
- プロトコル
  - OAuth2.0
  - OpenIDConnect を覚えておけばいい(拡張されているだけ)
- フロー
  - プロバイダに認証リクエストを送る
  - 認証が終わるとトークンリクエストを送る
  - 帰ってきた ID トークン(OpenID)を検証、必要に応じてアクセストークン(OAuth)を使う

OminiAuth の Wiki には以下の 2 ケースで記載してます。

- リクエストフェーズ
- コールバックフェーズ

具体的に何をしているのかのお話についてですが以下のことを説明してました。

- /auth/google_auth2 はなにしてる
  - リクエストフェーズの開始
  - OmniAuth::Storategies
  - request_phase メソッドがある
- call_back_phase メソッドがある
  - コールバックフェーズ
- 意外と OmniAuth はシンプル

「Tips」として。

- OmniAuth.conig.test_mode
  - 外部の ID プロバイダを使わなくてもテストできる
- SetupPhase がある
  - 認証フローの挙動を変更できる

### 所感

うちのシステムでも OmniAuth を管理画面ログインで導入しています。
私自身、この実装に携わっていないため、これがどのように動いているのか正直理解していませんでしたが、ざっくりとした全体像を掴むことができました。

今回を機に現実装の確認をしてみます。


## 約9000個の自動テストの時間を50分から10分に短縮、偽陽性率(Flakyテスト)を1％以下に抑えるまでの道のり
https://kaigionrails.org/2024/talks/hatsu38/

https://speakerdeck.com/hatsu38/yue-9000ge-nozi-dong-tesutono-shi-jian-wo50fen-10fen-niduan-suo-flakytesutowo1-percent-yi-xia-niyi-etahua

### 概要

- テストに 10 分以上かかってるプロジェクトが会場でも多かった
- 2023 年のテスト
  - 7800 テストぐらい
  - 50 分ぐらいかかる
  - Flaky なテストもたくさんある

やったことを以下に記載します。

- sleep が多い
  - have_text は wait するから削除できる
  - loading アイコンが消えたかをチェックするヘルパーの作成
- before_all も検討する
  - let_it_be
- parallel_test
  - テストファイルの分割
  - パラレルテストの仕組み
    - コード読んでみたい
- build が長い
- Flaky なテスト
  - ログやスクショの確認面倒
    - AllureReport の導入
        - https://allurereport.org/
  - capybara-playwight-driver の導入
    - 動画を確認可能
    - AllureReport でも確認可能
    - rails 7.1 で使える

やった結果、9200 のテストが増えたのに 10 分になったとのことです。

### 所感

自動テストの Flaky さは Hotwire にしたことで現在なくなったものの、今後もテストが増えていくことを考えたら時間が長くなってしまう可能性は高いはずです。
sleep をできるだけ使わないこと、失敗した時の確認がしやすくなるという方法は導入検討ありだなと思いました。
テストを動画で確認できる capybara-playwight-driver の導入もしてみたいと思いました。

あと、パラレルテストの仕組みはコードを読んでみたいです。


## Hotwire光の道とStimulus

https://kaigionrails.org/2024/talks/nay/
https://speakerdeck.com/nay3/hotwireguang-nodao-tostimulus

### 概要

- Hotwire 光の道(設計指針)
  - 誘導されるレールになってない
  - わかっていないと辛くなる
    - Stimilus コードが増える
    - DRY でないメンテしにくいコード
    - SPA の方が良かったみたいな
  - なにをわかればいい
    - サーバーサイド設計指針
      - 全部を画面遷移として考える
      - 紙芝居的
      - 処理を全部サーバーに
  - 大原則
    - 制御をサーバーに集める
    - 画面しか使うべきではない言葉なので修正してくださいことを作らない
  - クライアント側のはなし
    - Stimilus を選択肢にしない
    - JS のコードの共通化がむずかしい
    - 画面別にするケースが少ない
- Stimilus で何をやりたいか
  - Rails や Turbo でできないことをする
  - 何を担当するべき
    - Hotwire＋Rails と SPA はパラダイムが違う
    - React と Hotiwre を比較しがち
    - 同じレイヤーの代替物ではない
    - 基本は Rails が画面を変える
    - Rails とセットで React に対応している
  - 目的別
    - クリックによるイベント
      - 検索は submit するが正しい
      - 基本 submit でやる(click や onchange)
    - セレクトでの URL 遷移
      - 選択肢に対するなんかの URL に遷移すると考える
      - セレクトオプションに a タグとする
      - リンクがセットになっている
    - フォーカスを当てたい

- タブの選択
  - サーバーによらない画面変更はしない
  - 画面がオーガナイザーになってしまう
  - 紙芝居でやる(画面切り替え)



### 所感

かなり今回の発表は参考になりました。

今までは SPA 的な考えで、動的動きは JS を使ってしまうという考えを一旦棄却するべきでした。

紙芝居的な実装をすることで Stimulus は使わないで実装すること、使っても汎用的なコードで収まるようにすることの考え方が勉強になりました。


## Importmapを使ったJavaScriptの読み込みとブラウザアドオンの影響

https://kaigionrails.org/2024/talks/swamp09/
https://speakerdeck.com/swamp09/importmapwoshi-tutajavascriptno-du-miip-mitoburauzaadoonnoying-xiang


### 概要

- rails7 へのアップデート
  - Hotwire の導入
  - Webpacker の終わり
- 移行先
- importmap-rails ってなに
  - ライブラリのマッピング
  - モダンブラウザで使えるようになってる
- importmap がデフォルトになった理由
  - Babel のトランスパイルが不要になったから
  - ひとつの JS にまとめなくて良くなった(HTTP2 の導入により)
- pin で-- download オプションをつけると vender ディレクトリに入る
- V2 がダウンロードがデフォルト挙動になった

### 所感

うちはあまり importmap を使っていませんが、ライブラリを入れる際の注意点として参考になりました。

業務の話をすると、Sentry を JS 側に入れた方がいいかなと思いました。

## Data Migration on Rails

https://kaigionrails.org/2024/talks/ohbarye/
https://speakerdeck.com/ohbarye/data-migration-on-rails

### 概要

- data migration とは
  - 古いデータベースから新しいデータベースに移行すること
  - 単一のデータ変換、移動
- one persion framework
  - 一人の人間が使うべきではない言葉なので修正してくださいといけないことが多すぎる
  - 一人でも戦えるためのフレームワーク
  - datamigration は焦点当たってない
- data migration
  - スキーマを変更してデータを埋める
  - 望ましくないデータの解消
    - リカバリ対応

方法として以下を紹介してました。

- SQL での操作
  - bytebase などの saas がある(権限設定)
  - 整合性が取れなかったりする
- rails c、rails runner
  - validation などが実行される
  - 対話的に実行できる
- migration と一緒に実行
- Rake タスクなどと一緒に実行
  - テストも書ける
- 専用 GEM の活用
  - data-migrate ←これがいい
  - data_migrator
  - migration_data
  - rails-data-migraitons
  - active-data-miggrations
  - etc...
  - 学習コストが高いかも

https://github.com/Shopify/maintenance_tasks

また、rails generate script というコマンドが rails8 で使えるようになります。


### 所感

細かなデータマイグレーションをどのような方法でするかは、プロジェクトの状態や時間をどれだけかけることが可能かがポイントとなりそうでした。

maintenance_tasks は使ってみたい気持ちはあるものの、うちのプロジェクトでは現状そこまで必要とされない気がしております。

ただ、どのようなライブラリなのかは調べてみる価値はありそうでした。

## 30万人が利用するチャットをFirebase Realtime DatabaseからActionCableへ移行する方法

https://kaigionrails.org/2024/talks/ryosk7/

### 概要

- pato の開発
- 2017 年から Firebase を使ってた
  - us-central しか選べなかった
  - 1000 リクエスト/秒の制限

技術選定として。

- Firestore
  - 料金高い
  - 性能はその代わりいい
- Cloud pub/sub
  - メンテ不要
  - スループット、ストレージに料金がかかる
  - ベンダーロックがいや
- Action Cable
  - フロントの実装も楽
  - インフラのメンテ必要
  - 事例を作りたい
- dbmigratin はダウンタイムが発生するため、別テーブルに入れる
- flipper を利用


### 所感

別のデータベースに移行する作業はうちの業務と類似している部分がありました。

特に大量データに対し、db:migrate をするには少なからずダウンタイムが発生しうること。

そのためには、一度別テーブルに入れてまずはデータベース移行を完了させるという方法もひとつの方法として参考になりました。


## サイロ化した金融システムを、packwerk を利用して無事故でリファクタリングした話

https://kaigionrails.org/2024/talks/cc-kusama/

### 概要

- プロダクトが 10 を超えている
  - 認可ロジックが散らばってる
    - 横串の仕様がわかりにくい
    - 認可ロジックの妥当性確認コストが高い
- ロジックの集約
  - 影響範囲は大きい
  - モジュラーモノリスの採用
- packwerk の採用
  - コード移動のみでモジュールかできる
  - モジュラーモノリスかを支援する静的解析ツール
- 認可パッケージの作成し、処理の集約が可能となった
- 導入後の結果
  - 既存動作に影響を与えない設計にメリットあり
    - 本番環境に影響与えない
    - パッケージ化の安全・低コストで試せる
    - パブリックの設定ができる
- 防ぎきれない観点
  - 同じロジックが作成されることは防げない
    - 全体認知が必要(README など)


### 所感

マイクロサービスのようにアプリを完全に分離するのではなく、ディレクトリを分離する等、ロジックレベルで分割する手法は参考になる点が多かったです。

今のプロジェクトでは、新規実装がメインとなるため、リファクタリングによる packwerk を利用することはなさそうだが、調べてみたいです。


## 二日間の総括

二日にかけ、Kaigi on Rails に参加した感想をこちらにまとめます。


### よかったこと

普段関わることのできなかった技術やいつも関わっている技術の適切な使い方について知ることができ、新しい発見が多かったです。

前職のように社内で勉強会が定期的に開催されるわけではなく、いろいろな技術に触れる機会が激減していたため、改めてこのような場に参加できていろんなことを吸収できました。

またオフラインで参加し、場の雰囲気や空気感を感じつつ、同じように Ruby on Rails という技術に関わっている方々の話を聞くだけでもモチベーションが上がりました。

特に、Hotwire についての発表が 2 件ほどあったのですが、Hotiwre の使い方について SPA の考え方を捨てて Hotwire 的設計を考えなければならないのはとても参考になりました。

### いまいちだったこと

元々PHP や Laravel の業界で前職は仕事をしており、知り合いがいなかったことで心細かったところもありました。

東京のイベントいうことである程度のコミュニティーが形成され、一日目に懇親会があったのですが、あまり楽しむことができなかったです。

こういった場にはちょくちょく顔を出してコミュニケーションをとるような動きをした方がいいのかなと思いつつ、無理にそういったものに参加する必要もないだろうなという思いもあります。

## 最後に

少し声をかけられ、自分のことに聞かれることがあるのですが、取り組んでいる業務が少し異なるため、改めて聞かれると説明が難しいなと思いました。

それ w もあり、自分が少し特殊な業界に関わり周囲の人が関わっていない技術(AS400)を扱っているのだと思い知らされる場面がありました。

そんな特殊な業務について、自分も機会あれば外部の人にやっていることの紹介をしてみたいという気持ちが芽生えました。

どこまで今の業務を表に出せるのか次第ですが、登壇をいつかしてみたいという気持ちがつよくなり、そのためにいろんなことを今の業務から吸収できたらと思えるようになりました。
