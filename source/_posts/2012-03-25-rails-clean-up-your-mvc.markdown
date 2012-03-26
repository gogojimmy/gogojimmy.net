---
layout: post
title: "[Rails] 整理你的 MVC"
date: 2012-03-25 22:48
comments: true
categories: Rails
---

這是重構系列的文章：
* [[Rails] 設計模式：封裝](http://gogojimmy.net/Rails/rails-encapsulation/)
* [Rails] 整理你的 MVC

Rails 是一個 MVC 架構的框架這大家都知道，但初接觸的新手 (像我) 一定還是不知道 Code 邏輯該怎麼放比較好，因此我們很容易就會寫出像下面這樣的 Code：

``` ruby Index
<html>
  <body>
    <ul>
        <% User.find(:order => "last_name").each do |user| %>
        <li><%= user.last_name %> <%= user.first_name %></li>
        <% end %>
    </ul>
  </body>
</html>
```

在 View 中我們塞了一堆的邏輯，還包含了 ActiveRecord 的存取，這應該只有在 PHP 入門書才會看到這種程式，把 SQL 的存取都放在 View 中，HTML 就已經夠麻煩了我們實在不應該在裡面塞太多的邏輯，因此我們該考慮把邏輯放到別的地方，例如說上面的 Code 我們稍加改寫可能會是這樣：

``` ruby Index
<html>
  <body>
    <ul>
    <% @users.each do |user| -%>
    <li><%= user.last_name %> <%= user.first_name %></li>
    <% end %>
    </ul>
  </body>
</html>
```

``` ruby UsersController
class UsersController < ApplicationController
  def index
    @users = User.order("last_name")
  end
end
```
在這邊我們將邏輯的部份搬到了 Cotroller 中，現在 View 看起來就乾淨多了，但事實上，我們應該要把所有的類似於 order、where、slect 這一類的方法都搬到 Model層中，Rails 提供了 Scope 的方法讓我們可以很方便的建立出一些方法，例如上面的 Code 我們可以再改寫成這樣：

``` ruby UsersController
class UsersController < ApplicationController
  def index
    @users = User.ordered
  end
end
```

``` ruby Users
class User < ActiveRecord::Base
  scope :ordered, order("last_name")
end
```

好處一樣是為了先前提到的封裝性，我們不應該讓 Controller 知道太多實作的細節，假設一種情況：

``` ruby UsersController
class UsersController < ApplicationController
  def index
    @user = User.find(params[:id])
    @memberships =
      @user.memberships.where(:active => true).
      limit(5).
      order("last_active_on DESC")
  end
end
```

我們為了找出這個 user 的狀態為 active 的 memberships，在 Controller 中實作了 memberships 的部分便違反了封裝及 Law of Demeter，因此我們應該把這一類的 find 方法都搬到自己的 Model 中去實作，上面的 Code 我們應該這樣改寫：

``` ruby Users
class User < ActiveRecord::Base
  has_many :memberships
  scope :find_recent_active_memberships
    memberships.find_recently_active
  end
end
```

``` ruby Membership
class Membership < ActiveRecord::Base
  belongs_to :user
  scope :find_recently_active, where(:active => true).limit(5).order("last_active_on DESC")
end
```

而在 Controller 中就只剩下...

``` ruby UsersController
class UsersController < ApplicationController
  def index
    @user = User.find(params[:id])
    @recent_active_memberships = @user.find_recent_active_memberships
  end
end
```

這樣子 Membership 的實作便搬到了自己的 Model 中，假定今天我們想要尋找 user 所有狀態是 active 的 memberships，也只要在 Membership 的 Model 中建立一個 active 的 scope，我們就可以使用 `@user.memberships.active` 的方法就可調用，Controller 看起來就乾淨多了。

## 肥大的 Model

我們一直把這樣的邏輯搬到 Model 後，有一天你會發現自己的 Model 變得很肥大，MVC 有個概念是 [Skinny Controller, Fat Model](http://weblog.jamisbuck.org/2006/10/18/skinny-controller-fat-model)，但過於肥大的 Model 也是應該要重構一下的，例如下面這樣的例子：

``` ruby Order
class Order < ActiveRecord::Base
  def self.find_purchased
  end
  def self.find_waiting_for_review
  end
  def self.find_waiting_for_sign_off
  end
  def self.find_waiting_for_sign_off
  end
  def self.advanced_search(fields, options = {})
  end
  def self.simple_search(terms)
  end
  def to_xml
  end
  def to_json
  end
  def to_csv
  end
  def to_pdf
  end
end
```

在這個 Order Model 中我們可以看到大概分成三類的 method，find 類、search 類及 to_xxx 的轉檔類，在現實的 Coding 時只會更多不會更少，這樣的 Model 除了看起來肥大的難以閱讀之外，我們也應該思考一下 Model 的本質，什麼叫做 Model 的本質？意思就是說 Model 本身應該只要負責處理與自己相關的事物，無關的東西應該要被丟出去才對，在這個例子中像是 to_xxx 等的轉檔方法其實與 Order Model 本身並沒有關係，因此這一類方法其實應該要丟出去給別人處理，這裡的重構方法有很多種，這裡列出幾種手法：

###使用組合模式重構

重構結果會像這樣：

``` ruby Order
class Order < ActiveRecord::Base
  def converter
    OrderConverter.new(self)
  end
end
```

``` ruby OrderConverter
class OrderConverter
  attr_reader :order
  def initialize(order)
    @order = order
  end
  def to_xml
  end
  def to_json
  end
  def to_csv
  end
  def to_pdf
  end
end
```

上面的手法在設計模式中叫做『組合模式』，Order Model 是由一個 OrderConverter 的物件及其他所需要的物件所組合而成，Rails 本身所提供的 association methods(has_many, belongs_to...) 就都是由這樣的手法做出來的，使用上面的方法改寫後我們可以使用 `@order.converter.to_pdf` 這樣的方法來將 `@order` 轉成其他格式，但聰明如你我想一定發現了這樣的方法是違反 Law of Demeter 的，今天要是 converter 是 nil 的情況這行 code 就會炸掉，因此我們可以用之前學到的 delegate 來改寫：

``` ruby
class Order < ActiveRecord::Base
  delegate :to_xml, :to_json, :to_csv, :to_pdf, :to => :converter
  def converter
    OrderConverter.new(self)
  end
end
```

現在我們可以直接使用 `@order.to_pdf` 來將 order 轉成 pdf，並且將這些不是自己 Model 處理的 methods 全部委派出去，大幅縮減了 Model 本身。

### 使用 Module 重構
其實這應該算是最基本，也是最容易想到的手法，就是將這些行為類的方法包裝成 Module，然後在 Model 中 include 或是 extends 使用就好，就是以這個 Order Model 的例子，三種行為可以被拆成這樣：

``` ruby
module OrderStateFinders
  def find_purchased
  end
  def find_waiting_for_review
  end
  def find_waiting_for_sign_off
  end
  def find_waiting_for_sign_off
  end
end

module OrderSearchers
  def advanced_search(fields, options = {})
  end
  def simple_search(terms)
  end
end

module OrderExporters
  def to_xml
  end
  def to_json
  end
  def to_csv
  end
  def to_pdf
  end
end
```

我們將三種行為方法拆成三個不同的 Module，然後在我們的 Order Model 中可以這樣使用：

``` ruby
class Order < ActiveRecord::Base
  extend OrderStateFinders
  extend OrderSearchers
  include OrderExporters
end
```

縮減到只需要三行，多麼乾淨，這樣的作法不光光只是為了減少 Model 的程式碼而已，我們真正的目的是要讓 Model 本身處理真正與 Model 相關的重要事物，將不相關的程式碼搬到一些 Module 中會讓我們的 Model 更提高閱讀性，而被搬出去的方法甚至可以使用 Namespace 命名整理後加以重複使用，愉快。

有錯的地方歡迎指正！

這是重構系列的文章：
* [[Rails] 設計模式：封裝](http://gogojimmy.net/Rails/rails-encapsulation/)
* [Rails] 整理你的 MVC
