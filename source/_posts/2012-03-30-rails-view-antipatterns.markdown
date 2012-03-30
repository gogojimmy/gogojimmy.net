---
layout: post
title: "[Rails 重構] View-HTML 其實沒那麼難"
date: 2012-03-30 13:59
comments: true
categories: Rails, AntiPatterns, Refactor
---
##問題：不要學奇怪的 PHP

View 顧名思義在 MVC 的架構中擔任表現層的工作，Rails Developer 通常比較重視後端的工作像是 Model 及 Controller 的部份而去忽略了 View 的層面，在 Rails 中 View layer 泛指了像是 template、helper、javascript 及 CSS 的部份，Template 的部份通常會放在 `app/view/` 底下以子目錄的形式分別存著不同 Controller 的 不同 Action 以及 Mailer，其中還包含了 layout 的資料夾，用來放一些會共用的 layout，而像是 javascript 及 CSS 的檔案在 Rails 3.1 以後由於 Asset pipeline 的功能放置在 `app/assets/javascript` 及 `app/asset/stylesheets/` 底下，Rails 已經幫你規劃好這樣的結構怕你亂放你的 View，Rails 這樣硬性的 MVC 設定不同於 PHP 常會把複雜的邏輯搬到 View 的部份去實作，造成難以維護及測試的程式。

<!-- more -->

##重構: 使用內建的 Helper 來整理你的 View

Rails 本身提供了許多方便易用的 helper 讓我們在打造 View 的時候省下很多功夫，方便我們寫出簡單好維護的 View，例如在以前我們要寫個 update 的方法的 form 給 UsersController 的話可能是這樣的：

``` ruby
<%= form_for :user,
:url => user_path(@user),
:html => {:method => :put} do |form| %>
```

這樣的 Code 會產生出這樣的 HTML：

``` html
<form action="/users/5" method="post">
  <div style="margin:0;padding:0">
    <input name="_method" type="hidden" value="put" />
  </div>
```

而一個新增的 form 會長這樣：

``` ruby
<%= form_for :user, :url => users_path do |form| %>
```

而產生出這樣的 HTML：

``` html
<form action="/users" method="post">
```

注意到上面兩個例子其實有很多不必要的資訊，而且在 edit form 還必須指定 http request，實在是很麻煩，在 Rails 2.1 後我們只要統一的使用：

``` ruby
<%= form_for @user do |form| %>
```

Rails 會根據 @user 物件來判斷這個 form 要產生的格式，若這個 @user 是一個新建的物件尚未被儲存，那麼就會產生新增用的 POST 方法的 form，若是已經包含原有資訊的 @user 則產生 PUT 方法的 form 像下面這樣：

``` html
<form action="/users" method="POST" class="new_user" id="new_user">
```

及

``` html
<form action="/users/5" method="post" class="edit_user" id="edit_user_5">
  <div style="margin:0;padding:0">
    <input name="_method" type="hidden" value="put" />
  </div>
```

注意到這個版本還自動在 form_tag 幫你加上了 id 與 class，方便你做 javascript 的控制及 CSS 的 Style。

##重構：View 的邏輯該擺在 Helper 或是 Model?

當你要為了一些 View 的邏輯去寫 helper 的時候你必須注意你要寫的邏輯是否屬於 domain logic，那麼你就應該是把它放在 Model 中而非 Helper，例如今天要去編輯一個 POST 的時候有幾個條件要去判斷，而在欠缺考慮的情況下我們會寫成這樣：

``` ruby
<% if current_user &&
  (current_user == @post.user ||
  @post.editors.include?(current_user)) &&
  @post.editable? &&
  @post.user.active? %>
    <%= link_to 'Edit this post', edit_post_url(@post) %>
<% end %>
```

這樣的程式看起來就非常複雜，因此我們會希望能使用像是一個 `post_editable?` 的方法來判斷，但像是這樣的邏輯我們應該是搬到 Model 去做，這時候你的 View 看起來就會是這樣：

``` ruby
<% if @post.editable_by?(current_user) %>
	<%= link_to 'Edit this post', edit_post_url(@post) %>
<% end %>
```

一般來說，邏輯要被放在 Model 或是 Helper 取決在於這個邏輯的使用地點，以 `post_editable?` 這個方法來說將不僅僅被使用在 View 也可能會在 Controller 的 edit 和 update 方法中使用，那麼這方法理所當然的應該被擺在 Model 中，換句話說，要是這個方法僅僅會在 View 的部份被使用，就算他是與 Model 相關的方法，我們也會把它放在 Helper，例如說假設你今天有一個 Job 的 Model，用來產生一個 sidebar 的固定寬度的 widget 並會在上面列出一些 Job，有一天有個傢伙寫了 "Software Developer/Ruby/Taiwan"，中間包含了斜線造成不會自動換行因而讓你的 widget 破版，你嘗試了各種 HTML 及 CSS 的方法想要修正後發現唯一的辦法就只能在斜線中間加上空白讓他能換行。

以這個例子來說，你可以在 job 被儲存前就先對 title 去做處理以確保存進資料庫時的資料是合法的，但這樣的作法只能針對未來新增的資料，針對舊資料你必須去新增一個 migration 來撈出所有的舊資料來做處理，另外一種方法是去覆寫他的 getter，當 request 這個欄位的時候便處理從資料庫中撈出的資料將他格式化成期望的格式，這樣的方法解決了剛剛只能新增未來的問題，而且符合我們剛剛所說的只有在 View 中才會使用到這個方法，因此這個方法就非常的適合放在 Helper：

``` ruby
def display_title(job)
  job.title.split(/\s*\/\s*/).join(" / ")
end
```

##問題：看不懂的 HTML

具備語意的 Markup 具有容易維護及容易閱讀的優點，有著這三樣準則：

* 一個頁面中一個具有意義的區塊應該要有一個特定的 id 或是 class 去指向他。
* 使用正確的 HTML Tag 去 markup 頁面，例如說 `<p>` 用來顯示文字段落，而 `<h1>` 用來顯示主標題，而 `<div>` 與 `<span>` 本身是不具任何語意的。
* 任何的 Style 應該是在 CSS 中被完成而非在元素本身。

糟糕的例子：

``` html
<div>
  <div>
    <span style="font-size: 2em;">
      I love kittens!
    </span>
  </div>
  <div>
    I love kittens because they're
    <span style="font-style: italic">
      soft and fluffy!
    </span>
  </div>
</div>
```

修改過的例子：

``` html
<div id="posts">
  <div id="post_1" class="post">
    <h2>
      I love kittens!
    </h2>
    <div class="body">
      I love kittens because they're
      <em>
	soft and fluffy!
      </em>
    </div>
  </div>
</div>
```
一種判斷 markup 語意的方式是將內容全部抽離，只留下 markup，例如：

不好的例子：

``` html
<div>
  <div>
    <span style="font-size: 2em;" />
  </div>
  <div>
    <span style="font-style: italic" />
  </div>
</div>
```

修改過的例子：

``` ruby
<div id="posts">
  <div id="post_1" class="post">
    <h2/>
    <div class="body">
      <em/>
    </div>
  </div>
</div>
```

web design 最終的目標就是只要更改 CSS 的部分而不用去動到任何 HTML markup 就可以去呈現出一個新的樣式，例如 [http://csszengarden.com/](http://csszengarden.com/) 就是這樣的一個例子，這樣的目標首要之務就是讓你的 markup 保持語意性。一樣的觀念也適用於 javascript ，不應該使用行內的 javascript 而應該是把它抽離成單獨的檔案，讓我們的 HTML 版面維持簡單的 markup 就好。

假設你盡了全力想要讓你的 markup 看起來乾淨符合語意，跟你搭配的 Designer 跟你工作的笑呵呵給你個讚，經過了幾個小時的努力你刻出了下面的 markup：

``` ruby
<div class="post" id="post_<%= @post.id %>">
  <h2 class="title">Title</h2>
  <div class="body">
    Lorem ipsum dolor sit amet, consectetur...
  </div>
  <ol class="comments">
    <% @post.comments.each do |comment| %>
      <li class="comment" id="comment_<%= comment.id %>">
        <%= comment.body %>
      </li>
    <% end %>
  </ol>
</div>
```

然後你覺得累了，你覺得花了那麼多的時間來想這個部分的自己跟白痴一樣，你是一個後端的 Developer，你開始討厭寫網頁，你開始把責任丟給 Designer 然後等著他刻出糟糕的 markup 後再來吵架。其實這個問題可以使用 Rails 內建的 Helper 來幫助你省去很多時間，例如上面的 Code 經過使用 Helper 的改寫後會變這樣：

``` ruby
<%= div_for @post do %>
  <h2 class="title">Title</h2>
  <div class="body">
    Lorem ipsum dolor sit amet, consectetur...
  </div>
  <ol class="comments">
    <% @post.comments.each do |comment| %>
      <%= content_tag_for :li, comment do %>
        <%= comment.body %>
      <% end %>
    <% end %>
  </ol>
<% end %>
```

使用 `div_for` 這個 helper 來取代原本的手刻 div，直接就會幫你建立出具備語意的 markup，讓你省時之餘也能更容易的以 Rails 的方式來 markup，這樣的方式也集中統一了 class 與 id，當你今天 CSS 想要更動所有 Post 的 class 或 id 時就非常方便，其他像是最常使用的 `form_for` 的 Helper 也有相同的作用，這讓負責前端的 Designer 或是 Javascript developer 可以方便的使用 id 及 class 來指向他們需要的部份。使用這些 Helper 有助於幫助你快速的打造出具備語意的 markup。

##重構：使用 Haml 及 Sass

HTML 很麻煩，因此有人想出了 HTML 的模版 Haml，例如今天你有個複雜的 HTML：

``` html
<div id="blawg">
  <%= div_for @post do %>
    <h2 class="title">
      <%= @post.title %>
    </h2>
    <div class="body">
      Lorem ipsum dolor sit amet, consectetur...
    </div>
    <ol class="comments">
      <% @post.comments.each do |comment| %>
        <%= content_tag_for :li, comment do %>
        <%= comment.body %>
      <% end %>
      <% end %>
    </ol>
  <% end %>>
</div>
```

看起來就非常頭痛，如果是 Haml 的版本：

``` haml
div#blawg
  div for @post
    h2.title
      output @post.title
    div.body
      "Lorem ipsum dolor sit amet, consectetur..."
    ol.comments
      for each @post.comments as comment
    li for comment
      output comment.body
```

Haml 使用縮排來表示階層關係，使用像是 jQuery 的 Selector 來表示 id 與 class，不用寫 close tag，你甚至可以拋棄 div ，Haml 會很聰明的幫你加上去：

``` haml
#blawg
  %div[@post]
    %h2.title= @post.title
  .body
    Lorem ipsum dolor sit amet, consectetur...
  %ol.comments
    - @post.comments.each do |comment|
      %li[comment]
        = comment.body
```

Saas 是 Haml 的姊妹，是屬於 CSS 的 Markup，在 Rails 3.1 以後加入了 Asset pipeline 後變成了寫 CSS 的預設工具，非常之好用阿！！有空的話再為他寫一篇吧。

##心得：
我想在初學 Rails 的時候最容易犯的錯誤就是把邏輯放在 View 中了，像一般 PHP 的入門書教學那樣甚至會把 SQL Query 放在 View 中的毫無架構可言，而寫 Rails 的時候最常思考的問題就是複雜的邏輯到底要擺哪去，我們的目的其實最終就是為了寫出好維護及好閱讀的 Code ，如此而已，因此架構性的思考是非常重要的。

另外除了像是 Haml 這個 Template 外，其實還有像是 [Slim](http://slim-lang.com/index.html) 也是非常不錯的樣板引擎(但我沒用過，希望有用過的人可以分享一下XD)，CSS 的部份也有個 [LESS](http://lesscss.org/) 跟 SASS 日月爭輝，雖然我沒用過 LESS，但是 SASS 可是非常推崇好用的阿，搭配 [Compass](http://compass-style.org/) 後就根本是天下無敵了。
