---
layout: post
title: "[Rails] 設計模式：封裝"
date: 2012-03-24 14:10
comments: true
categories: Rails
---
寫 Rails 也有一段時間了，目前在 ihower 的建議下開始讀 [Rails AntiPatterns](http://www.amazon.com/Rails-AntiPatterns-Refactoring-Addison-Wesley-Professional/dp/0321604814) 這本書，練習重構自己的程式碼，接下來我想將是一系列的讀書心得。

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

## Law of Demeter
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

## Delegate

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
