---
title: "Todoアプリでmorphingを使ってみる"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rails", "ruby", "hotwire"]
published: true
publication_name: "yamaden"
---

本記事では、Rails 8・Hotwire・TailwindCSS を使用し、Todo アプリを作成します。

また、Todo の作成・編集・削除を本来 Hotwire では Turbo Stream を使って変更していきますが、こちらについて morphing を使って実装置き換えまでしていきます。


## プロジェクトのセットアップ

まず、新しい Rails プロジェクトを作成します。Rails 8 では、TailwindCSS のサポートが組み込まれているため、`--css`オプションで簡単に設定できます。

```zsh
rails new todo_app --css tailwind
cd todo_app
```

## モデルの作成

Todo モデルを作成します。title、description、completed の 3 つの属性を持たせます。

```zsh
rails generate model Todo title:string description:text completed:boolean
rails db:migrate
```

モデルにバリデーションと便利なスコープを追加します。

```ruby
# app/models/todo.rb
class Todo < ApplicationRecord
  validates :title, presence: true

  after_initialize :set_defaults, if: :new_record?

  scope :recent, -> { order(created_at: :desc) }
  scope :completed, -> { where(completed: true) }
  scope :incomplete, -> { where(completed: false) }

  private

  def set_defaults
    self.completed ||= false
  end
end
```

## ビューの作成

### インデックスページ

```erb
# app/views/todos/index.html.erb
<div class="max-w-4xl mx-auto px-4 py-8">
  <h1 class="text-3xl font-bold mb-8">Todos</h1>

  <%= turbo_frame_tag "todo_form" do %>
    <div class="bg-white shadow-sm rounded-lg p-6 mb-8">
      <h2 class="text-xl font-semibold mb-4">新しいTodoを作成</h2>
      <%= render "form", todo: @todo %>
    </div>
  <% end %>

  <div class="bg-white shadow-sm rounded-lg">
    <div class="p-6">
      <h2 class="text-xl font-semibold mb-4">Todo一覧</h2>
      <%= turbo_frame_tag "todos" do %>
        <div class="space-y-4">
          <%= render @todos %>
        </div>
      <% end %>
    </div>
  </div>
</div>
```

一覧画面には、新しい Todo が追加できるフォームを準備し、その下に Todo リストが表示されるようなデザインとします。

turbo_frame_tag で `todo_form`と`todos`を括っておきます。

form は、次項にて説明する partial file を作成しておきます。

### Formのpartial file

```erb
# app/views/todos/_form.html.erb
<%= form_with(model: todo, class: "space-y-4") do |form| %>
  <% if todo.errors.any? %>
    <div class="bg-red-50 p-4 rounded-lg">
      <div class="text-red-700 font-medium">
        <%= pluralize(todo.errors.count, "個のエラー") %>が発生しました:
      </div>
      <ul class="list-disc list-inside text-red-600">
        <% todo.errors.full_messages.each do |message| %>
          <li><%= message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= form.label :title, "タイトル", class: "block text-sm font-medium text-gray-700 mb-1" %>
    <%= form.text_field :title, class: "block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500" %>
  </div>

  <div>
    <%= form.label :description, "説明", class: "block text-sm font-medium text-gray-700 mb-1" %>
    <%= form.text_area :description, rows: 3, class: "block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500" %>
  </div>

  <div class="flex items-center">
    <%= form.check_box :completed, class: "h-4 w-4 rounded border-gray-300 text-indigo-600 focus:ring-indigo-500" %>
    <%= form.label :completed, "完了", class: "ml-2 block text-sm text-gray-700" %>
  </div>

  <div class="flex justify-end">
    <%= form.submit "保存", class: "inline-flex justify-center rounded-md border border-transparent bg-indigo-600 py-2 px-4 text-sm font-medium text-white shadow-sm hover:bg-indigo-700 focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:ring-offset-2" %>
  </div>
<% end %>
```

### Todoのpartial file

リストの partial file はこちらです。

```erb
# app/views/todos/_todo.html.erb
<%= turbo_frame_tag todo do %>
  <div class="bg-white border rounded-lg shadow-sm hover:shadow-md transition-shadow duration-200">
    <div class="p-4">
      <div class="flex items-start justify-between">
        <div class="flex items-start space-x-3 flex-grow">
          <%= form_with(model: todo, class: "flex items-center") do |form| %>
            <%= form.check_box :completed,
                class: "h-4 w-4 rounded border-gray-300 text-indigo-600 focus:ring-indigo-500",
                onchange: "this.form.requestSubmit()" %>
          <% end %>

          <div class="flex-grow">
            <h3 class="text-lg font-medium <%= todo.completed? ? 'line-through text-gray-500' : 'text-gray-900' %>">
              <%= todo.title %>
            </h3>
            <p class="mt-1 text-sm text-gray-500">
              <%= todo.description %>
            </p>
          </div>
        </div>

        <div class="flex items-center space-x-2 ml-4">
          <%= link_to edit_todo_path(todo),
              class: "text-gray-400 hover:text-gray-500" do %>
            <svg class="h-5 w-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z" />
            </svg>
          <% end %>

          <%= button_to todo_path(todo),
              method: :delete,
              class: "text-gray-400 hover:text-red-500",
              data: { turbo_confirm: "このTodoを削除してもよろしいですか？" } do %>
            <svg class="h-5 w-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
            </svg>
          <% end %>
        </div>
      </div>
    </div>
  </div>
<% end %>
```

リストの中でも turbo_frame_tag `todo`で括っておきます。

ただし、ここの`todo`はそれぞれの todo オブジェクトを示してます。

### Editのerb


```erb
<%= turbo_frame_tag @todo do %>
  <%= render "form", todo: @todo %>
<% end %>
```

Todo の partial の編集ボタンを押したら、この erb が呼ばれます。

しかし、turbo_frame_tag を使っていますので、Todo の partial file の turbo_frame_tag と置き換わる仕様になります。

## コントローラーの実装

Todos コントローラーに CRUD 操作と Turbo Streams のサポートを実装します。

```ruby
# app/controllers/todos_controller.rb
class TodosController < ApplicationController
  before_action :set_todo, only: [:show, :edit, :update, :destroy]

  def index
    @todos = Todo.order(created_at: :desc)
    @todo = Todo.new
  end

  def create
    @todo = Todo.new(todo_params)

    respond_to do |format|
      if @todo.save
        format.turbo_stream { render turbo_stream: turbo_stream.prepend("todos", partial: "todos/todo", locals: { todo: @todo }) }
        format.html { redirect_to todos_path, notice: "Todoが作成されました。" }
      else
        format.turbo_stream { render turbo_stream: turbo_stream.replace("todo_form", partial: "todos/form", locals: { todo: @todo }) }
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end

  def update
    respond_to do |format|
      if @todo.update(todo_params)
        format.turbo_stream { render turbo_stream: turbo_stream.replace(@todo, partial: "todos/todo", locals: { todo: @todo }) }
        format.html { redirect_to todos_path, notice: "Todoが更新されました。" }
      else
        format.turbo_stream { render turbo_stream: turbo_stream.replace("todo_#{@todo.id}", partial: "todos/todo", locals: { todo: @todo }) }
        format.html { render :edit, status: :unprocessable_entity }
      end
    end
  end

  def destroy
    @todo.destroy

    respond_to do |format|
      format.turbo_stream { render turbo_stream: turbo_stream.remove(@todo) }
      format.html { redirect_to todos_path, notice: "Todoが削除されました。" }
    end
  end

  private

  def set_todo
    @todo = Todo.find(params[:id])
  end

  def todo_params
    params.require(:todo).permit(:title, :description, :completed)
  end
end
```


## Hotwireの設定

Hotwire は既に Rails 8 にデフォルトで含まれていますが、Flash メッセージの自動消去などの追加機能を実装するために、Stimulus コントローラーを作成します。

```javascript
// app/javascript/controllers/removable_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["container"]

  remove() {
    this.containerTarget.remove()
  }
}
```

## スタイリングの適用

アプリケーション全体のレイアウトを設定し、TailwindCSS のユーティリティクラスを活用します。

```erb
# app/views/layouts/application.html.erb
<!DOCTYPE html>
<html class="h-full bg-gray-50">
  <head>
    <title>TodoMorph</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= stylesheet_link_tag "tailwind", "inter-font", "data-turbo-track": "reload" %>
    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
  </head>

  <body class="h-full">
    <div class="min-h-full">
      <nav class="bg-indigo-600">
        <div class="mx-auto max-w-7xl px-4 sm:px-6 lg:px-8">
          <div class="flex h-16 items-center justify-between">
            <div class="flex items-center">
              <div class="flex-shrink-0">
                <h1 class="text-white text-xl font-bold">TodoMorph</h1>
              </div>
            </div>
          </div>
        </div>
      </nav>

      <% if notice.present? %>
        <div class="bg-green-50 p-4" data-controller="removable" data-removable-target="container">
          <div class="flex">
            <div class="flex-shrink-0">
              <svg class="h-5 w-5 text-green-400" viewBox="0 0 20 20" fill="currentColor">
                <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd"/>
              </svg>
            </div>
            <div class="ml-3">
              <p class="text-sm font-medium text-green-800"><%= notice %></p>
            </div>
          </div>
        </div>
      <% end %>

      <main>
        <div class="mx-auto max-w-7xl py-6 sm:px-6 lg:px-8">
          <%= yield %>
        </div>
      </main>

      <%= turbo_frame_tag "modal" %>
    </div>
  </body>
</html>
```

ここまで作成すると以下のような画面が作成されます。

![todo-hotwire](/images/todo-hotwire.png)

Turbo Stream は、id の部分を target として置き換えたり更新したりする仕組みです。

参考：https://turbo.hotwired.dev/reference/streams


## controllerのリファクタリング

ここで少し改善をしていきます。

先ほどの create メソッドを再掲します。

```rb
  def create
    @todo = Todo.new(todo_params)

    respond_to do |format|
      if @todo.save
        format.turbo_stream { render turbo_stream: turbo_stream.prepend("todos", partial: "todos/todo", locals: { todo: @todo }) }
        format.html { redirect_to todos_path, notice: "Todoが作成されました。" }
      else
        format.turbo_stream { render turbo_stream: turbo_stream.replace("todo_form", partial: "todos/form", locals: { todo: @todo }) }
        format.html { render :new, status: :unprocessable_entity }
      end
    end
  end
```

`format.turbo_stream`としていますが、`todos/create.turbo_stream.erb`を設置することで自動判別可能です。


```erb
<%= turbo_stream.prepend("todos", partial: "todos/todo", locals: { todo: @todo }) %>
```

ここで turbo_stream.prepend メソッドについて詳しくみていきます。

Ref：https://rubydoc.info/github/hotwired/turbo-rails/Turbo%2FStreams%2FTagBuilder:prepend


Hotwire の Turbo Stream としては以下のような Element が生成できれば OK です。

```html
<turbo-stream action="prepend" target="todos">
  <template>
    Example
  </template>
</turbo-stream>
```

使い方としては以下です。

```rb
<%= turbo_stream.prepend "clearances", "<div id='clearance_5'>Prepend this to .clearances</div>" %>
<%= turbo_stream.prepend "clearances", clearance %>
<%= turbo_stream.prepend "clearances", partial: "clearances/unique_clearance", locals: { clearance: clearance } %>
<%= turbo_stream.prepend "clearances" do %>
  <div id='clearance_5'>Prepend this to .clearances</div>
<% end %>
```

ですので、先ほどの`todos/create.turbo_stream.erb`は以下のように書くことも可能です。

```erb
<%= turbo_stream.prepend "todos", @todo %>
```

controller はシンプルになります。

```rb
  def create
    @todo = Todo.new(todo_params)
    unless @todo.save
      render :new, status: :unprocessable_entity
    end
  end
```

同じように destroy も変えていきます。


元ソース。

```rb
  def destroy
    @todo.destroy

    respond_to do |format|
      format.turbo_stream { render turbo_stream: turbo_stream.remove(@todo) }
      format.html { redirect_to todos_path, notice: "Todoが削除されました。" }
    end
  end
```

`todos/destroy.turbo_stream.erb`を作成します。

```erb
<%= turbo_stream.remove @todo %>
```
controller もシンプルになります。

```rb
  def destroy
    @todo.destroy
  end
```

update も同じです。


`todos/update.turbo_stream.erb`を作成し、controller をシンプルにします。


```erb
<%= turbo_stream.replace @todo %>
```


```rb
  def update
    unless @todo.update(todo_params)
      render :edit, status: :unprocessable_entity
    end
  end
```


ここまでやると Turbo Stream での実装が完了します。

## 件数の追加

todo の件数を入れたいとなります。

```erb
<div class="max-w-4xl mx-auto px-4 py-8">
  <h1 class="text-3xl font-bold mb-8">Todos</h1>

  <%= turbo_frame_tag "todo_form" do %>
    <div class="bg-white shadow-sm rounded-lg p-6 mb-8">
      <h2 class="text-xl font-semibold mb-4">新しいTodoを作成</h2>
      <%= render "form", todo: @todo %>
    </div>
  <% end %>

  <div class="bg-white shadow-sm rounded-lg">
    <div class="p-6">
      <h2 class="text-xl font-semibold mb-4">Todo一覧</h2>
      <%= turbo_frame_tag "todos" do %>
        <div class="space-y-4">
          <%= render @todos %>
        </div>
      <% end %>
    </div>
    <p id="total_count" class="m-4">件数：<%= @todos.count %></p> ←追加
  </div>
</div>
```

ただし、こうしてしまうと Todo 削除時にこの件数が更新されず不整合状態となります。

そうならないよう、件数も update が必要となります。

`todos/destroy.turbo_stream.erb`に以下を追記します。

```erb
<%= turbo_stream.remove @todo %>
<%= turbo_stream.update "total_count" do %>
  <p class="m-4">件数：<%= @todo_count %></p>
<% end %>
```


```rb
  def destroy
    @todo.destroy
    @todo_count = Todo.count
  end
```

こうすることで件数の部分の`id="total_count"`のところが新しい件数で update されるようになります。

また、create の時も件数が増えるべきなので、`todos/create.turbo_stream.erb`を変更します。


```erb
<%= turbo_stream.prepend "todos", @todo %>
<%= turbo_stream.update "total_count" do %>
  <p class="m-4">件数：<%= @todo_count %></p>
<% end %>
```

```rb
  def create
    @todo = Todo.new(todo_params)
    unless @todo.save
      render :new, status: :unprocessable_entity
    end
    @todo_count = Todo.count
  end
```

ここではやっていませんが、件数部分を partial 化しておくと良いです。


## Turbo Streamの課題

ここまでやってきて気になったはずです。

件数の追加のように部分的更新が増えれば増えるほど複雑性が増していきます。

なので、できれば Turbo Stream は使わない方がコードはシンプルに保てるはずです。

そこで Turbo Drive に立ち返りたいですが、どうしてもスクロール部分が Top に戻ってしまうという懸念があります。

そこで Turbo Drive の「morphing」を使ってみましょう。

参考：https://turbo.hotwired.dev/reference/events#page-refreshes


### morphingの設定

設定としては大まかに以下です。

- layouts ファイルの head タグに`<%= yield :head %>`を埋め込む
- controller からはリダイレクトで返す
- erb ファイルに`<% turbo_refreshes_with(method: :morph, scroll: :preserve) %>`を埋め込む

これだけでスクロール位置を保持して対応できます。

なので、Turbo Stream のコードは削除していきましょう。

まずは controller のリダイレクトに書き換えます。

```rb
  def create
    @todo = Todo.new(todo_params)
    unless @todo.save
      render :new, status: :unprocessable_entity
    end
    redirect_to todos_path
  end

  def update
    unless @todo.update(todo_params)
      render :edit, status: :unprocessable_entity
    end
    redirect_to todos_path
  end

  def destroy
    @todo.destroy
    redirect_to todos_path
  end
```

`todos/_todo.html.erb`の削除ボタンですが、turbo_frame_tag の中にあるボタンのため、Turbo Drive にするため、target を top にします。

```erb
          <%= button_to todo_path(todo),
              method: :delete,
              class: "text-gray-400 hover:text-red-500",
              data: { turbo_confirm: "このTodoを削除してもよろしいですか？", turbo_frame: "_top" } do %>
```

Turbo Stream は使用しないため、以下は削除します。

- `todos/create.turbo_stream.erb`
- `todos/update.turbo_stream.erb`
- `todos/destroy.turbo_stream.erb`

最終的な commit はこちらです。

https://github.com/Naoki0722/todo_sample/commit/29a3103db08564635535b6cf1356e870e45b8b39

これによって Turbo Drive を基本とし、スクロール位置を保持してくれます。


## Hotwireの考え方

とある記事でこのように記載ありました。

>See, your life gets more complex whenever you add partial updates to the mix. Now you have to care about screen regions, the elements they contain, and how interactions affect them. Good abstractions help, but you can’t shake the additional complexity off. You are just in a more complex realm.
>This is why we say that Turbo is progressive: go with the happy Turbo Drive path by default — and deviate from it when you need higher fidelity for specific screens or interactions.

参考：https://dev.37signals.com/a-happier-happy-path-in-turbo-with-morphing/

日本語訳しました。

>画面の一部だけを更新する仕組み（部分更新）を取り入れると、システム全体がより複雑になります。
>画面の領域や、その中に含まれる要素、さらにそれらに対するインタラクションがどのように影響を及ぼすかまで気にしなければならなくなるからです。
>優れた抽象化（設計）でこれを助けることはできますが、この追加された複雑さを完全に取り除くことはできません。
>それは単に、より複雑な領域に足を踏み入れるということなのです。
>だからこそ、私たちは Turbo を「漸進的（progressive）」だと表現しています。
>基本的には、Turbo Drive による「ハッピーなパス（理想的なルート）」をデフォルトとして使用し、特定の画面やインタラクションにおいてより高精度な動作が必要な場合にのみ、そのデフォルトから逸脱するべきだと考えています。

つまり、できるだけ Turbo Stream は使わない方が開発としてもやりやすいということです。


## まとめ

今回、Turbo Stream で実装したものを敢えて Turbo Drive に戻しました。

部分更新といっても Turbo Drive で対応可能であることを知り、できるだけシンプルに実装していくのがコツかなと考えてます。

参考リポジトリ：https://github.com/Naoki0722/todo_sample
