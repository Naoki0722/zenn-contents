---
title: "作成・編集機能をTurbo への実装置き換え"
emoji: "🗒️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hotwire", "rails"]
published: true
publication_name: "yamaden"
---

ここからは前回の記事で Hotwire と Rails をインストールしたプロジェクトで、DM 機能を実装しましたがその実装を Hotwire に置き換えていきます。

## DMの編集機能実装(Non Hotwire)

その前にやりたいこととして、DM で送信したメッセージを編集する機能を実装しておきます。

これは実際に不要なのですが、Turbo frames を説明するためにも実装しておきます。

`MessagesController`は作成済みのため、`update`, `edit`メソッドを作成し編集できるようにします。

config/routes.rb

```rb
resources :messages, only: %i[new create]
↓
resources :messages, only: %i[new create edit update]
```

とりあえず編集画面に遷移できるように編集ボタンを追加、controller のメソッドの追加。

app/views/rooms/show.html.erb:14-15

```rb
<%= link_to_if message.own_message?(current_user), '編集', edit_room_message_path(@room.id, message.id),
    class: 'bg-blue-500 hover:bg-blue-700 text-white py-2 px-4 rounded' %>
```


app/controllers/messages_controller.rb

```rb
  def edit
    @room = Room.find(params[:room_id])
    @message = Message.find(params[:id])
  end

  def update
    message = Message.find(params[:id])
    if message.update(save_params)
      flash[:success] = "Message updated!"
      redirect_to room_path(message.room)
    else
      flash[:error] = message.errors.full_messages.join(", ")
      redirect_to edit_room_message_path(message.room, message)
    end
  end
```

app/views/messages/edit.html.erb

```rb
<h2 class="text-2xl">メッセージ編集画面</h2>

<%= form_with(model: @message, url: room_message_path, class: 'my-8') do |form| %>
  <%= form.hidden_field :room_id, :value => @message.room_id %>
  <%= form.hidden_field :user_id, :value => @message.user_id %>
  <div>
    <%= form.label :content, "送信内容" %>
    <%= form.text_area :content, rows: 5, class: "border rounded p-2 w-full" %>
  </div>
  <div>
    <%= form.submit "編集する", class: "bg-blue-500 text-white py-2 px-4 rounded" %>
  </div>
<% end %>
```

この時点で編集ボタンをクリックしたら DM の編集画面に遷移ならびに編集が可能となります。

この編集機能は通常の Rails の遷移で実現された実装になっております。

## 編集機能をTurbo Framesへ置き換え

ここからは実際に編集機能を Turbo Frames に置き換えていきます。

まず、置き換えに伴い編集フォームについては、DM の内容と送信ボタンを横に並べたようなデザインとし、DM ルームで置き換わっても違和感ないように変更します。

app/views/messages/edit.html.erb

```rb
<%= form_with(model: @message, url: room_message_path, class: 'my-8') do |form| %>
  <%= form.hidden_field :room_id, :value => @message.room_id %>
  <%= form.hidden_field :user_id, :value => @message.user_id %>
  <div class="flex items-start justify-end">
    <%= form.text_area :content, class: "border rounded p-2 mr-4 w-3/5" %>
    <%= form.submit "編集する", class: "bg-blue-500 text-white py-2 px-4 rounded" %>
  </div>
<% end %>
```

こうすることでフォームとボタンが横並びになり、ルームページで表示しても違和感なくなりました。

次に frame_id を設定していきます。

frame_id を設定することでその id が一致したものが入れ替わるということになります。

まず、 app/views/rooms/show.html.erb の個別メッセージに `turbo_frame_tag`を設定します。

```rb
<%= turbo_frame_tag message do %>
  <div class="my-8 <%= message.own_message?(current_user) ? 'text-right' : 'text-left' %>">
    <p class="mb-1">
      <strong><%= message.user.name %></strong>
      <span class="text-gray-500 text-sm"><%= message.created_at.to_fs %></span>
      <p class="my-2"><%= message.content %></p>
      <%= link_to_if message.own_message?(current_user), '編集', edit_room_message_path(@room.id, message.id),
      class: 'bg-blue-500 hover:bg-blue-700 text-white py-2 px-4 rounded' %>
    </p>
  </div>
<% end %>
```

そして、 app/views/messages/edit.html.erb にも `turbo_frame_tag`を設定します。


```rb
<%= turbo_frame_tag @message do %>
  <%= form_with(model: @message, url: room_message_path, class: 'my-8') do |form| %>
    <%= form.hidden_field :room_id, :value => @message.room_id %>
    <%= form.hidden_field :user_id, :value => @message.user_id %>
    <div class="flex items-start justify-end">
      <%= form.text_area :content, class: "border rounded p-2 mr-4 w-3/5" %>
      <%= form.submit "編集する", class: "bg-blue-500 text-white py-2 px-4 rounded" %>
    </div>
  <% end %>
<% end %>
```

こうすることで、show.html.erb でクリックした編集ボタンは同じ`turbo_frame_tag`の id を探しにいき、コンテンツを置き換える動作をします。

`turbo_frame_tag message`や `turbo_frame_tag @message`としてるのは、turbo_frame_tag に渡す引数２モデルを渡すと暗黙的に id とします。

### バリデーションエラー対応

バリデーションエラーに対応できるように改修をします。


まずエラーが起きたら edit を render するようにします。

```rb
render :edit, status: :unprocessable_entity
```

その edit.html.erb は下記のようにフォームエラーを表示するようにします。

```rb
<%= turbo_frame_tag @message do %>
  <%= form_with(model: @message, url: room_message_path, class: 'my-8') do |form| %>
    <%= form.hidden_field :room_id, :value => @message.room_id %>
    <%= form.hidden_field :user_id, :value => @message.user_id %>
    <div class="flex items-start justify-end">
      <div class="w-3/5 mr-4 ">
        <%= form.text_area :content, class: "border rounded p-2 w-full" %>
        <span class="text-red-500"><%= @message.errors.full_messages_for(:content).presence&.join(',') %></span>
      </div>
      <%= form.submit "編集する", class: "bg-blue-500 text-white py-2 px-4 rounded" %>
    </div>
  <% end %>
<% end %>
```

content の text_area には`field_with_errors`クラスが付与されるため、css で調整します。


```css
.field_with_errors {
  @apply contents;
}

.field_with_errors textarea {
  border: 2px solid red;
  background-color: #ffe6e6;
}
```

こうすることでバリデーションエラーが起きた場合フォームは赤くなり、エラー内容が表示されます。

### 成功時の処理

現状の controller は、下記の通りです。

```rb
  def update
    @message = Message.find(params[:id])
    if @message.update(save_params)
      redirect_to room_path(@message.room)
    else
      render :edit, status: :unprocessable_entity
    end
  end
```

このままルームページにリダイレクトするので更新後の値が表示されるはずです。

flash はいらないので削除しました。

ここまでくるとインラインで DM を編集できるようになるはずです。

## DMの新規送信をTurbo Streamで実装

ここからはすでに実装された DM 送信を Turbo Stream を使って実装していきます。

流れとしては以下です。

- ルームページで新規作成ボタンを押したら登録フォームがインラインで表示される
- 新規作成を押すとルームページの DM に追加される

新規ボタンを押すことで、messages_controller の new メソッドが呼ばれ、new.html.erb が呼ばれます。

これを DM ページで表示させるようにします。

まず、新規作成ボタンで以下の通りとしました。

```rb
<%= turbo_frame_tag 'new_message' %>
<div class="mt-24">
  <%= link_to "メッセージを作成", new_room_message_path(@room.id), class: 'bg-green-500 hover:bg-green-700 text-white font-bold py-2 px-4 rounded',
      data: {turbo_frame: 'new_message'} %>
  <%= link_to "リストに戻る", rooms_path, class: 'bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded' %>
</div>
```

link_to メソッドに data 属性の `turbo_frame: 'new_message'`を指定しています。

こうすることでリンクをクリックした際、turbo_frame の new_message を探しにいきます。

新規ボタンを押すことで、messages_controller の new メソッドが呼ばれます。
その後、new.html.erb が呼ばれる erb には、以下のように turbo_frame_tag を`new_message`で指定してます。

そうすることで、show.html.erb の`new_message`と置き換えるようになります。

```rb
<%= turbo_frame_tag 'new_message' do %>
  <%= form_with(model: @message, url: room_messages_path, class: 'my-8') do |form| %>
    <%= form.hidden_field :room_id, :value => @message.room_id %>
    <%= form.hidden_field :user_id, :value => @message.user_id %>
    <div>
      <%= form.label :content, "送信内容" %>
      <%= form.text_area :content, rows: 5, class: "border rounded p-2 w-full" %>
    </div>
    <div>
      <%= form.submit "送信する", class: "bg-blue-500 text-white font-bold py-2 px-4 rounded" %>
      <%= link_to 'キャンセル', room_path(@message.room_id), class: 'ml-2 border border-black py-2 px-4 rounded' %>
    </div>
  <% end %>
<% end %>
```

つぎに、`新規作成を押すとルームページの DM に追加される`実装をするのですが、messages_controller の create メソッドが呼ばれます。

ただし、今回は DM のリストに新規追加したものをリストとして増やしたいのと、入力フォームは消したいので、2 箇所を更新する必要があります。

そこで、turbo_frame ではなく、turbo_stream を利用します。


messages_controller の create メソッドは以下のようにします。

```rb
  def create
    @message = Message.new(save_params)
    if @message.save
      flash[:success] = "Message sent!"
      # redirect_to room_path(@message.room)
    else
      flash[:error] = @message.errors.full_messages.join(", ")
      redirect_to new_room_message_path(@message.room)
    end
  end
```

変更点としては、これまでは作成が成功したら DM ルームへリダイレクトしていましたがこれを辞めます。

その代わりに、`create.turbo_stream.erb`を作成します。

```rb
<%= turbo_stream.append "messages" do %>
  <%= render partial: 'rooms/list' %>
<% end %>

<%= turbo_stream.update "new_message", '' %>
```

- `messages`に新規追加したものを append する処理
- `new_message`は入力フォームを非表示にするように update する処理

こうすることで登録時にリスト追加され、フォームは非表示となります。

また、バリデーションに失敗した場合、flash の:error にエラーメッセージを追加します。
new_room_message_path にリダイレクトするようしていますが、ここのリダイレクトはやめます。


```rb
    else
      # flash[:error] = @message.errors.full_messages.join(", ")
      # redirect_to new_room_message_path(@message.room)
      render :new, status: :unprocessable_entity
    end
```

new.html.erb は下記のようにすることでエラーメッセージを確認しつつ、フォームを赤く表示可能です。

```rb
<%= turbo_frame_tag 'new_message' do %>
  <%= form_with(model: @message, url: room_messages_path, class: 'my-8') do |form| %>
    <%= form.hidden_field :room_id, :value => @message.room_id %>
    <%= form.hidden_field :user_id, :value => @message.user_id %>
    <div>
      <%= form.label :content, "送信内容" %>
      <%= form.text_area :content, rows: 5, class: "border rounded p-2 w-full" %>
      <span class="text-red-500"> ←追加
        <%= @message.errors.full_messages_for(:content).presence&.join(',') %> ←追加
      </span> ←追加
    </div>
    <div>
      <%= form.submit "送信する", class: "bg-blue-500 text-white font-bold py-2 px-4 rounded" %>
      <%= link_to 'キャンセル', room_path(@message.room_id), class: 'ml-2 border border-black py-2 px-4 rounded' %>
    </div>
  <% end %>
<% end %>
```

以上で作成と編集を Hotwire に置き換えました。
