---
layout: post
title: "[Rails 重構] Model(1)-談談封裝"
date: 2012-03-24 14:10
comments: true
categories: Rails, Refactor, AntiPatterns
published: false
---
寫 Rails 也有一段時間了，目前在 ihower 的建議下開始讀 [Rails AntiPatterns](http://www.amazon.com/Rails-AntiPatterns-Refactoring-Addison-Wesley-Professional/dp/0321604814) 這本書，練習重構自己的程式碼，接下來我想將是一系列的讀書心得。

<!--more-->

##問題：雜亂無章的 Model
由於 Rails 本身是一個已經設計好的框架，我們在寫 Code 時不會像是 Java 般的需要考慮設計模式方面的問題，久而久之也容易寫出一些設計不好的 Code，物件導向的產生就是為了維持程式的彈性及可維護性，因此我們在寫 Code 時通常必須為了日後好維護而使用一些手法去包裝，例如下面這種情況：

``` ruby
class Address < ActiveRecord::Base
  belongs_to :customer
end

class Customer < ActiveRecord::Base
  has_one :address
  has_many :invoices
end

class Invoice < ActiveRecord::Base
  belongs_to :customer
end
```

我們在這裡有三個 Model，一個 Customer 擁有一個 Address 與很多的 Invoices，我們在設計 Invoice 的 View 時可能就會有這種情況：

``` ruby
<%= @invoice.customer.name %>
<%= @invoice.customer.address.street %>
<%= @invoice.customer.address.city %>,
<%= @invoice.customer.address.state %>
<%= @invoice.customer.address.zip_code %>
```

Rails 本身內建的 Association 非常好用，因此我們可能就會寫出像上面這樣的例子，我們為了存取到這張 Invoice 的客戶地址，我們便使用了 `@invoice.customer.address` 這樣的方法，但假設今天在 Addess 的部份有所更動，你會發現你必須去更改所有使用 `@invoice.customer.address` 呼叫的地方，簡單來說就是程式耦合度過高，還有一種常見的情況是如果中間的物件是 nil 的話這時候 Code 就會爆炸，非常容易在寫測試的時候跳出 NoMethod Error 的錯誤，因此我們必須開始考慮重構。

##重構：Law of Demeter
物件導向中有個特性叫封裝，封裝的目的就是要降低類別間互相依賴的程度，那就牽扯到『能見度』的問題，如果怕另外一個類別太依賴於自己的實作，那麼就不要讓他知道我們實作的內容，將自己的資料隱藏起來，僅提供一個對外的 API 這就是所謂的『資訊隱藏』(Information Hiding)。一個封裝的法則叫做 [Law of Demeter](http://en.wikipedia.org/wiki/Law_of_Demeter)，又稱為『最少知識原則』(Principle of Least Knowledge)，具體來說，一個物件 O 以及他的方法 M 只能夠調用以下幾種物件的方法：

  1. O 自己本身。
  2. M 的參數。
  3. 任何由 M 建立或初始的物件。
  4. O 的直接組合物件。

簡單來說，當你去買 100 元的東西時，店員只要知道他最後能拿到 100 元就好，而並不需要知道這 100 元是怎麼來的，你從口袋拿出來，或是錢包拿出來，都跟他沒有關係，如果今天程式寫成 `店員.顧客.錢包(100)` 那麼今天要是顧客錢是放在口袋不就不能付錢了，因此 Law of Demeter 還有個原則是『只使用一個點』(Use only one dot)，像是 `a.b.method()` 就違反了 Law of Demeter，而 `a.method()` 則不會，例如說剛剛的例子我們可以這樣重構：

``` ruby
class Customer < ActiveRecord::Base
  has_one :address
  has_many :invoices
  def street
    address.street
  end
  def city
    address.city
  end
  def state
    address.state
  end
  def zip_code
    address.zip_code
  end
end

class Invoice < ActiveRecord::Base
  belongs_to :customer
  def customer_name
    customer.name
  end
  def customer_street
    customer.street
  end
  def customer_city
    customer.city
  end
  def customer_state
    customer.state
  end
  def customer_zip_code
    customer.zip_code
  end
end
```

比較一下之前的例子，現在我們已經將 Address 的部份在 Customer 中完成了，我們的 Invoice 現在只要使用 `customer_street` 這個方法就可以存取到 `Customer` 中的 `Address` 類別屬性。

``` ruby
@invoice.customer.name => @invoice.customer_name
```
符合了 Law of Demeter 中『僅使用一個點』的原則，今天如果 Address 的類別有所變動，也僅需要在 Customer 中自己處理而不會影響到 Invoice 類別，但是寫到這邊你一定還是會想，這樣子我寫的 Code 感覺好像沒有少多少，如果今天要變動 Address 的時候，在 Customer 中的 Code 也沒少到哪去，而且其實有些方法搞不好根本用不上，是不是還有改進的地方？

##重構：使用 delegate

Rails 提供了我們一個很方便的方法 delegate 能幫我們達到上面一樣的效果，例如說上面的 Invoice 和 Customer 我們可以這樣改寫：

``` ruby
class Customer < ActiveRecord::Base
  has_one :address
  has_many :invoices
  delegate :street, :city, :state, :zip_code, :to => :address
end

class Invoice < ActiveRecord::Base
  belongs_to :customer
  delegate :name,
           :street,
           :city,
           :state,
           :zip_code,
           :to => :customer,
           :prefix => true
end
```

很明顯的可以看出來，在 Customer 中我們將一些方法 delegate 給 Address Model，而在 Invoice 中也將一些方法 delegate 給 Customer，並且加上了 `:prefix => true `的參數來產生 `customer_` 的 prefix，因此我們在 Invoice 的 instance variable 就可以使用 ‘@invoice.customer_street‘ 這樣的方法，我們一樣維持了 Law of Demeter，將物件資訊對外封裝，這樣的寫法跟上面落落長的 Code 是等價的，如果你還加上 `:allow_nil => true` 的參數的話表示像是在呼叫 `@invoice.customer_street` 時，customer 是 nil 的時候不會跳出 NoMethod Error 而是回傳 nil，感謝 Rails。

##重構：整理你的 find()

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

好處一樣是為了先前提到的封裝性，我們不應該讓 Controller 知道太多實作的細節。

##重構：封裝，還是封裝

假設一種情況：

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

##問題：笨重的 Model

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

##重構：使用組合模式將方法拋出去

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

##重構：使用 Module
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

##心得
Model 算是 Rails 一個非常重要的部份，因為他包含了大部分的邏輯，當然也因為包含了大部分的邏輯因此使得 Model 容易變得很笨重，寫 Rails 時最常考慮的問題就是希望寫出來的 Code "易懂"，因此我們要在 Model 中下不少功夫來盡量達到這樣的需求。
