---
layout: post
title: "[Ruby 重構]Refactoring from Good to Great 影片心得"
date: 2012-11-13 22:53
comments: true
categories: Ruby, Refactor, Rails
---

今天看了一支*Ruby Conf 2012*的影片，標題是[Refactoring from Good to Great](http://www.confreaks.com/videos/1233-aloharuby2012-refactoring-from-good-to-great)，主講人是[Ben Orenstein](http://www.confreaks.com/presenters/780-ben-orenstein)，非常強烈推薦去看一下影片，裡面所提到重構技巧非常實用，以下是為影片做的一些筆記：

<!--more-->

首先我們先來看看這段程式：

``` ruby
class OrdersReport
  def initialize(orders, start_date, end_date)
    @orders = orders
    @start_date = start_date
    @end_date = end_date
  end

  def total_sales_within_date_range
    orders_within_range =
    @orders.select { |order| order.placed_at >= @start_date &&
      order.placed_at <= @end_date }
    orders_within_range.
    map(&:amount).inject(0) { |sum, amount| amount + sum }
  end
end

def Order < OpenStruct
end
```

我們有一個`Order`以及作為報表的`OrderReport`兩支*Class*，在初始化一個`OrderReport`物件時必須傳入`orders`、`start_time`及`end_time`三個參數，其中`total_sales_within_date_range`這個方法會計算出給定的時間區間中所有的訂單總數。

看起來很簡單，程式也運作正常，但這出了什麼問題？首先我們把目光放到`total_sales_within_date_range`這個方法：

``` ruby
def total_sales_within_date_range
  orders_within_range =
  @orders.select { |order| order.placed_at >= @start_date &&
    order.placed_at <= @end_date }
    orders_within_range.
    map(&:amount).inject(0) { |sum, amount| amount + sum }
end
```

在這個方法中，定義了一個`orders_within_range`的變數，用`select`方法篩選出了區間的訂單，再來將這些訂單的數量做加總的動作，你會發現在這個方法中我們擁有了很多邏輯上的實作，這不是件好事，不好在過兩個月後你來看你的程式，或是你的同事來看你的程式時，他無法馬上看懂你在做什麼，以一個*Class*的*public*方法而言他擁有了太多實作的細節，比較下面這個版本：

``` ruby
def total_sales_within_date_range
  orders_within_range.
  map(&:amount).inject(0) { |sum, amount| amount + sum }
end

private

def orders_within_range
  @orders.select { |order| order.placed_at >= @start_date &&
    order.placed_at <= @end_date }
end
```

這個版本我們將`orders_within_range`拉出去作為一個*private*的方法，這樣有什麼好處？作為一個*Class*的*public*方法原則就是越簡單越好，細節越少越好，因為這個是外面的其他人會去使用的方法，在物件導向中稱之為最小知識原則(*Least Knowledge Principle*)，對於外面使用*API*的人他沒有必要知道邏輯細節的實作，因此我們把它藏成*private*的方法，再者，你現在回頭看原先`total_sales_within_date_range`這個方法，是不是比原先的寫法容易理解許多，我們從字面上可以知道`orders_within_range`這個變數會幫我取得訂單，然後我再去做運算即可，非常好，我們完成了第一步，也是很重要的一步。

再來我們看到我們剛剛被拉出去做成*private*的`orders_within_range`這個方法，在這邊我們用`>=`及`<=`來比對日期區間與訂單日期以求得訂單，問題在於這樣的寫法還不夠好，除了不夠直覺之外，對於篩選出某個區間的訂單這個方法似乎不該屬於報表類別的責任，我們可以試著改寫成下面的版本：

``` ruby
def orders_within_range
  @orders.select { |order| order.placed_between?(@start_date, @end_date)
end

def Order < OpenStruct
  def placed_between?(start_date, end_date)
    placed_at >= start_date && placed_at <= end_date
  end
end
```

這個版本中我們將原先的比對拉到了`Order`類別去做，找出一個範圍的訂單這件事情本來就應該屬於訂單的責任而不是報表，我們包裝成一個簡單的`placed_between？`的方法來進行比對，因此在原先的`orders_within_range`方法中就可以呼叫這個方法來使用，這樣看起來是不是比原先的程式看起來又更直覺了些？

下一步我們回頭看看我們全部的程式，你會發現到處都有著`start_date`以及`end_date`，而且兩者還缺一不可，這告訴我們這兩個是著高度依賴性必須同時存在的變數，這個情形稱之為[Data Clumps](http://sourcemaking.com/refactoring/data-clumps)，這時候你應該要把這兩個或是多個變數抽出來成為一個物件來處理，我們試著改寫這段程式：

``` ruby
class OrdersReport
  def initialize(orders, date_range)
    @orders = orders
    @date_range = date_range
  end

  def total_sales_within_date_range
    orders_within_range.
    map(&:amount).inject(0) { |sum, amount| amount + sum }
  end

  private

  def orders_wihin_range
    @orders.select { |order| order.placed_between?(@date_range)
  end
end

class DateRange < Struct.new(:start_date, :end_date)
end
```

你看到這邊最重要的是，我們宣告了一個簡單的`Struct`類別包含了`start_date`以及`end_date`，原本的`OrderReport`類別中`initialize`方法需要傳入的參數從三個減少成兩個，所有原先需要`start_date`以及`end_date`的地方現在全部都只有一個`date_range`物件來取代，最簡單來說，我們用了一個有意義的名字來代表了一些複雜的事情，另一個好處在於我們再降低了`OrderReport`以及`DateRange`的耦合性，一個只需要傳入兩個參數的類別可能造成的耦合一定比要傳入三個參數的類別來的小，如同上面所說，一個類別所需要知道的內容越少，他的耦合度就越低，彈性也就越高。再者，被封裝成物件的`date_range我們現在還可以去為他定義一些行為，我們現在回頭看看`Order`這個類別的`placed_between?`這個方法，其中有著日期比對的判斷，這理所當然要屬於`DateRange`應該去做的事情，因此我們再改寫一下我們的程式：

``` ruby
def DateRange < Struct(:start_date, :end_date)
  def include?(date)
    (start_date..end_date).cover?(date)
  end
end

def Order < OpenStruct
  def placed_between?(date_range)
    date_range.include?(placed_at)
  end
end
```

我們將日期判斷這件事情移到了`DateRange`裡面去做，在這邊我們改使用了`cover`方法來判斷是否符合，`cover`方法不同於`include`方法會將整個陣列元素取出到記憶體，`cover`僅找出陣列的頭尾元素並加以判斷，這樣程式看起來就分工分的很好。

現在我們來重構最後一個部分，我們來看看`total_sales_within_date_range`這個方法，在這方法我們先取出了`orders_within_range`，然後計算出總數，看起來已經不錯了，但我們還可以再做一步，試著將計算的部份也抽出來看看會有什麼結果：

``` ruby
class OrdersReport
  def initialize(orders, date_range)
    @orders = orders
    @date_range = date_range
  end

  def total_sales_within_date_range
    total_sales(orders_within_range)
  end

  private

  def total_sales(orders)
    orders.map(&:amount).inject(0, :+)
  end

  def orders_wihin_range
    @orders.select { |order| order.placed_between?(@date_range)
  end

end
```

你看到了我們將計算的方法也拆成一個`private`的方法，比對原來的部分我們的`public`方法只剩下了`total_sales_within_date_range`你一眼就可以看出來這個*public*的內容在做什麼，而裡面的相關邏輯實作並不是最重要的且並不是我們想知道的，因此我們把它封裝成`private`，你的`public`方法現在看起來非常乾淨且簡單，並不存在當初的複雜邏輯，`public`的部份是與外部溝通的，它本來就必須看起來很簡單，取用它的人並不需要知道你的邏輯細節實作，因此我們才將它封裝成`private，現在這看起來就是一個非常易懂且實用的類別！

再來我們看到下一個範例，一個`JobSite`類別，其中有著必填的`location`參數以及選填的`contact`：

``` ruby
class JobSite
  attr_reader :contact
  def initialize(location, contact)
    @location = location
    @contact = contact
  end

  def contact_name
    if contact
      contact.name
    else
      "no name"
    end
  end

  def contact_phone
    if contact
      contact.phone
    else
      "no phone"
    end
  end

  def email_to_contact(contact)
    contact.deliver_personalized_email(email_body)
  end
end

class Contact < OpenStruct
  def deliver_personalized_email(email)
    email.deliver(name)
  end
end
```

因為`contact`不是必填，因此我們定義了一些方法當`contact`沒有傳入的時候則回傳"no xxx"，你會發現整個`Class`每次去要資料的時候都會對`contact`做一次是否存在的判斷，我們可以加以改善：

``` ruby
class JobSite
  attr_reader :contact
  def initialize(location, contact)
    @location = location
    @contact = contact || NullContact.new
  end

  def contact_name
    contact.name
  end

  def contact_phone
    contact.phone
  end

  def email_to_contact(contact)
    contact.deliver_personalized_email(email_body)
  end
end

class NullContact
  def name
    "no name"
  end

  def phone
    "no phone"
  end

  def deliver_personalized_email(body)
  end
end

class Contact < OpenStruct
  def deliver_personalized_email(email)
    email.deliver(name)
  end
end
```

看出來了嗎？我們當`initialize`方法中的`contact`沒有傳入的時候，我們製造一個新的`NullContact`物件，這個物件擁有與`Contact`一樣的方法，但是會回傳所有的預設值，這時候我們就可以將原先程式中那些擾人的判斷給一一砍掉，多美妙阿，我們在呼叫的時候再也不需要知道現在這個`contact`是否存在，伴隨而來的壞處是，現在你必須讓`Contact`與`NullContact`兩個*API*保持同步，不然就會出錯。

再來我們看到最後一個範例，*User*使用一些第三方金流進行付款或退款時的處理：

``` ruby
class User
  SUBSCRIPTION_AMOUNT = 10.to_money

  def charge_for_subscription
    braintree_id = BraintreeGem.find_user(email).braintree_id
    BraintreeGem.charge(braintree_id, SUBSCRIPTION_AMOUNT)
  end

  def create_as_customer
    BraintreeGem.create_customer(email)
  end
end

class Refund
  def process!
    transaction_id = BraintreeGem.find_transaction(order.braintree_id)
    BraintreeGem.refund(transaction_id, amount)
  end
end
```

在`User`中使用了`BraintreeGem`的*API*來做付款動作，你會發現我們必須在`User`中做一些與`User`不相關的事情，例如你必須找出`braintree_id`、`transaction_id`然後對他進行charge的動作，這樣對嗎？`User`不應該去知道這些細節的實作，再者，如果今天在付款方式有所更動，我們想要除了`BraintreeGem`之外另外增加付款方式，而這時要去更改`User`不是很奇怪的事情嗎？因此我們應該將它放到應該放的地方：

``` ruby
class PaymentGateway
  SUBSCRIPTION_AMOUNT = 10.to_money

  def initialize(gateway = BraintreeGem)
    @gateway = gateway
  end

  def charge_for_subscription(user)
    braintree_id = BraintreeGem.find_user(email).braintree_id
    @gateway.charge(braintree_id, SUBSCRIPTION_AMOUNT)
  end

  def create_as_customer
    @gateway.create_customer(email)
  end

  def refund(refund_model)
    transaction_id = @gateway.find_transaction(order.braintree_id)
    @gateway.refund(transaction_id, amount)
  end
end

class User
  def charge_for_subscription
    PaymentGateway.new.charge_for_subscription(self)
  end

  def create_as_customer
    PaymentGateway.new.create_as_customer(self)
  end
end

class Refund
  def process!
    PaymentGateway.new.refund(self)
  end
end
```

我們新建立了一個`PaymentGateway`物件，用來處理所有跟交易有關的事務，並且將預設的*gateway*指定為`BraintreeGem`，如此我們的`User`現在不需要知道那些實作的細節，也不用知道到底是用什麼付款機制，它所需要做的就是呼叫付款的*API*，這多麼美好，這讓我們更好去測試所有我們想要測試的行為，因為所有的行為經過如此的封裝後更為簡單單純，*User*的部份我們只需測是否有呼叫到*API*，而在`PaymentGateway`的部份就測試相關的付款行為，不會再糾結在一起，這就是一個好設計。

## 什麼時候該進行重構？

幾個作為判斷的提示：

1. 找出你的*God Object*，幾乎所有的*App*都會有兩個最重要的*Class*，其中一個幾乎都是*User*，另一個就是你的*App*的主題，例如以電子商務來說就像是*Order*或是*Product*。找出*God Object*有兩種方式，找出所有*Model*的行數，最多行的那幾個肯定需要重構。
2. 找出更動最頻繁的那些*Model*(High churn files)，更動頻繁的檔案意味著你的*API*寫的不夠好因此頻繁的被更改，[churn](https://github.com/danmayer/churn)這個*Gem*能幫助你找出這些檔案。
3. 第三就是找出很常發生*Bug*的部分，那表示那邊有太複雜的邏輯因此你才會出現*Bug*。

最後[Ben Orenstein](http://www.confreaks.com/presenters/780-ben-orenstein)推薦了三本重構必讀聖經：

1. [Clean Code: A Handbook of Agile Software Craftsmanship (Paperback)](http://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882):這是寫出簡單直覺且乾淨的程式碼的入門書。
2. [Refactoring: Improving the Design of Existing Code (Hardcover)](http://www.amazon.com/Refactoring-Improving-Design-Existing-Code/dp/0201485672/ref=sr_1_1?s=books&ie=UTF8&qid=1352823235&sr=1-1&keywords=refactoring+improving+the+design+of+existing+code):必買，這是聖經。
3. [Growing Object-Oriented Software, Guided by Tests (Paperback)](http://www.amazon.com/Growing-Object-Oriented-Software-Guided-Tests/dp/0321503627/ref=sr_1_1?s=books&ie=UTF8&qid=1352823314&sr=1-1&keywords=growing+object-oriented+software+guided+by+tests)裡頭提到了非常多實用的*OO*觀念以及*TDD*技巧。

最後，如果有時間，建議妳可以看看影片：

{% youtube DC-pQPq0acs %}
