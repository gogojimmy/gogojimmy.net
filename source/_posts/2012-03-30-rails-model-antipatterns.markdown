---
layout: post
title: "[Rails 重構] Model(3)-或許你根本並不需要一個 Model"
date: 2012-03-30 13:58
comments: true
categories: Rails, Refactor, AntiPatterns
---
Rails 本身提供了 ActiveRecord 這套好用的 ORM，讓我們省去很多時間去處理繁瑣的資料細節，但是其實我們在設計一個 Web Application 的時候常會忽略了 Rails 本身提供的好用功能而去過度設計，這個章節的目的就是要避開這些過度的設計。

<!-- more -->

##問題：過度預期的設計

一個情況像是在設計使用者權限的時候，我們可能會為使用者建立多種角色擁有不同的權限，程式看起來會是這樣：

``` ruby
class User < ActiveRecord::Base
  has_and_belongs_to_many :roles, :uniq => true
  def has_role?(role_in_question)
    self.roles.first(:conditions => { :name => role_in_question }) ?
      true : false
  end
  def has_roles?(roles_in_question)
    roles_in_question =
      self.roles.all(:conditions => ["name in (?)",
                                     roles_in_question])
      roles_in_question.length > 0
  end
  def can_post?
    self.has_roles?(['admin',
                     'editor',
                     'associate editor',
                     'research writer'])
  end
  def can_review_posts?
    self.has_roles?(['admin', 'editor', 'associate editor'])
  end
  def can_edit_content?
    self.has_roles?(['admin', 'editor', 'associate editor'])
  end
  def can_edit_post?(post)
    self == post.user ||
      self.has_roles?(['admin', 'editor', 'associate editor'])
  end
end
```

在這個 User Model 中，一個 user 擁有很多種不同的 roles，其中單數的 has_role? 及複數的 has_roles? 方法定義了判斷該 user 是否有一個或是多個的角色權限，預期這些不同的角色可以去執行不同權限的行為，在這個 Model 中也提供了幾種不同的行為來讓擁有該權限的角色的用戶來執行。

這個 User Model 有幾個重大的缺陷，首先 has_role? 這個方法根本從頭到尾都沒有用到，反而是複數的 has_roles 方法在這個 Model 中被使用到，而且我們相信在這個 Application 中 has_roles 的方法將會被廣泛的使用，因此單數的 has_role 便是一個出自預期但其實不需要的方法。

第二則是這些 can_xxx 的方法，首先他提供了一個混亂且容易 confuse 的介面來使用，你可能永遠不知道有多少的 can_xxx 方法可以使用，再者在很多情況這些方法可能是來自於你的推測想像說以後 Application 可能會用到，因此你在需求產生前便先寫出來，但他可能永遠用不到。最後總結就是使用了像這樣的方法去硬刻出每種不同的權限行為，如果今天有個方法更動可能你就要找出整個 Application 更動的地方來更改，更慘的是若是他失效了你可能還不知道。

而 Role 的 Model 可能長得是這樣：

``` ruby
class Role < ActiveRecord::Base
  has_and_belongs_to_many :users
  validates_presence_of :name
  validates_uniqueness_of :name
  def name=(value)
    write_attribute("name", value.downcase)
  end
  def self.[](name) # Get a role quickly by using: Role[:admin]
    self.find(:first, :conditions => ["name = ?", name.id2name])
  end
  def add_user(user)
    self.users << user
  end
  def delete_user(user)
    self.users.delete(user)
  end
end
```

這個 Role Model 也存在著幾個問題：首先在這支程式中根本沒有能提供管理者新增或移除角色的功能，因此覆寫 name 的 setter 看起來就是個問題，再來是覆寫 role 的 getter 希望他像個 hash 般的作用，但一樣的毛病就是他完全沒有被使用到，這也是個沒有需求而只是為了方便而產生的方法。最後我們看到 add_user 及 remove_user 這兩個方法，用來讓為使用者增加或刪除 role，但這並不是一個好設計，因此，他在這支程式中也並沒有被使用到。

以上兩支 Model 的總結就是，有太多的方法是在產生需求前就憑空想像出來，而打造出沒有被使用的方法，這樣的想法就會讓你寫出像上面的『過度預期』的程式。

##重構：使用簡單的 Boolean 欄位取代

我們首先來改造一下我們的 User Model，用一種最簡單的方式：

``` ruby
class User < ActiveRecord::Base
end
```

很乾淨吧，沒錯就是把它清光光，這樣一來你就擺脫了 Role Model 的糾纏，我們可以簡單的在 User Table 使用三個 Boolean 欄位分別是 Admin、Editor、Writer，這樣 Rails 就會自動的提供你 admin?、editor?、writer? 三個方便的方法讓你用來判斷該 user 是否具有三種 role 的權限，在產生一個 user 的時候也只需要簡單的 checkbox 介面就可以完成 role 的指定，何必寫那麼多東西呢您說是不，沒有程式就是最好的程式。

但你說，未來如果我還需要增加很多不同的 Roles 的話，那豈不是就增加很多個 Boolean 的欄位？這樣看起來其實挺笨的，因此如果你真的有這樣的需求，其實不妨把 Role Model 再加回來，但是我們不使用剛剛所使用的 `has_and_belongs_to_many` 的方法，而是簡單的使用 `has_many: roles`，而在 Role Model 中我們使用一個常數陣列來記錄不同的角色：

``` ruby
class User < ActiveRecord::Base
  has_many :roles
end

class Role < ActiveRecord::Base
  TYPES = %w(admin editor writer guest)
  validates :name, :inclusion => { :in => TYPES }
  class << self
    TYPES.each do |role_type|
      define_method "#{role_type}?" do
        exists?(:name => role_type)
      end
    end
  end
end
```

這裡我們使用了 metaprogramming 的技巧，將 TYPES 陣列中的每個角色名稱都跑一次並建立出像是 admin?、editor? 這樣的方法來使用，因此我們可以使用 `user.roles.admin?` 這樣的方法，你甚至也可以把 metaprogramming 的部份搬到 User Model 中來實作，像這樣：

``` ruby
class User < ActiveRecord::Base
  has_many :roles
  Role::TYPES.each do |role_type|
    define_method "#{role_type}?" do
      roles.exists?(:name => role_type)
    end
  end
end
class Role < ActiveRecord::Base
  TYPES = %w(admin editor writer guest)
  validates :name, :inclusion => {:in => TYPES}
end
```

這樣你就可以使用 `@user.admin?` 的方法，跟原本使用 Boolean 的方式一樣，一個會較有爭議的地方是有些人會認為你把 Role 的相關實作搬到了其他的 Model(User)，但以這情況來說 `@user.admin?` 這樣的方法會比起 `@user.roles.admin?` 來的更直觀，因此這樣的方式其實是由你個人喜好，重點是我們用這樣的方式解決了先前『過度預期』設計而產生了一堆不必的方法的問題，並且提供了一個直觀且簡單的介面讓所有地方都可使用，而不必去記一些像是 `can_xxx` 的方法。

##結論：

像上面的概念所影響的不僅僅是這個 Model，而是整支程式所有可能會使用到 Model 方法判斷的地方，下面幾個問題提供你判斷

* 不要在需求產生之前就寫出那個 method。
* 如果需求還不明確，就先不要寫。
* 不要直接的考慮使用 Model，通常可能會有像是 Boolean 或是 Denormalization 的解法來避免產生過多的 Model。
* 如果沒有 CRUD 的需求，那麼通常不需要使用 Model，一個簡單的 Boolean 欄位、hash、array 或許就能解決問題了。

##問題：太多的 Model

過度的使用 Model 會使我們的程式變得龐大且複雜，很多人常想都不想的就新增一個 Model 來解決事情，但你必須了解當你新增一個 Model 的同時，意味著你增加了一個 Database 的 Migration、一些 Model 的單元測試、和一些需要的 Finders 及 Validations。

##重構：打破正規化

``` ruby
class Article < ActiveRecord::Base
  belongs_to :state
  belongs_to :category
  validates :state_id, :presence => true
  validates :category_id, :presence => true
end
class State < ActiveRecord::Base
  has_many :articles
  class << self
    all.each do |state|
      define_method "#{state}" do
        first(:conditions => { :name => state })
      end
    end
  end
end
class Category < ActiveRecord::Base
  has_many :articles
end
```

以這個例子來說，當我們想要設定一個 Article 的 State 時可能會這麼做：

``` ruby
@article.state = State.published
```

而檢查狀態可能就是：

``` ruby
@article.state == State.published
```

當然從我們學過的技巧中你可能也會把 metaprogramming 這一塊搬到 Article 去用以使用像是 @article.published? 這樣的方法來使用，不管怎樣，像這樣的狀況經常發生，我們為了一個物件的狀態而去新增了一個 Model 來管理他，以這個例子來說，其實我們在日後不太可能也不太應該會去新增刪除 State，換句話說一個不太需要新增刪除的資料其實是不用被儲存在資料庫的，因此其實我們根本不需要一個 Model 來處理他，因此我們便可以使用上一篇所使用的 denormalize 技巧使用一個陣列來儲存即可。

##重構：使用 Rails 的 Serialize

一個表單的使用者介面範例像是這樣：

![form](https://lh4.googleusercontent.com/-vaj6N0hDh8c/T3SKXugpPvI/AAAAAAAABGc/9IvNQy-rLmU/s800/RailsAntiPatternsBestPracticeRubyonRailsRefactorin.jpg)

這是一個基本的表單讓我們能夠填寫我們從哪邊得知訊息，當你選擇 "others" 的時候也必須在下面的欄位填寫內容，這樣的需求非常普遍，在 Model 的部份有幾種作法可以參考：

###Has and Belongs to Many

第一種作法就是將這些欄位使用一個 Referral 的 Model 包裝起來，然後使用 `has_and_belongs_to_many` 的方法來建立關係：

``` ruby
class User < ActiveRecord::Base
  has_many :referral_types
end
class Referral < ActiveRecord::Base
  has_and_belongs_to_many :users
end
```

除此之外 User 本身需新增一個字串的屬性，用來儲存當使用者選擇 "Other" 的時候所填寫的理由。

###Has Many

第二種作法是使用 `has_many` 來建立他們的關係：

``` ruby
class User < ActiveRecord::Base
  has_many :referral_types
end
class Referral < ActiveRecord::Base
  VALUES = ['Newsletter', 'School', 'Web',
            'Partners/Events', 'Media', 'Other']
  validates :value, :inclusion => {:in => VALUES}
  belongs_to :user
end
```

當然 User Model 也必須和上面一樣新增一個欄位用來填寫當使用者選擇 "Other" 時的資料。

### 什麼都別做

上面兩個作法都是使用了另一個 Model 來處理，我們回想一下之前所提到的信條，這個 Refferal Model 是否真的有存在的必要？他似乎沒有什麼 CRUD 的需求，我們很少會需要去新增或是刪除用戶從哪邊聽到這個訊息吧！因此實踐這個章節的信條，我們也可以不使用 Model 來達到一樣的功能，有兩種方法可以做到，其一是先前提過的我們可以在 User 中增加 Boolean 的欄位用來儲存這些選項，這方法完全合用，但在 Table 中增加這六個 Boolean 看起來實在有點蠢，因此這邊我們要介紹 Rails 提供的方法： `serialize`，改寫的 Code 像是這樣：

``` ruby
class User < ActiveRecord::Base
  HEARD_THROUGH_VALUES = ['Newsletter', 'School', 'Web',
                          'Partners/Events', 'Media', 'Other']
  serialize :heard_through, Hash
end
```

`serialize` 這個方法可以幫助我們將像是 Array、Hash 等 non-mappable 的資料序列化後儲存起來，以這個例子來說我們使用 `serialize` 的方法宣告了 `:heard_through` 這個欄位為一個 Hash 的形式，因此我們可以在 View 的部分這樣使用：

``` ruby
<%= fields_for :heard_through, (form.object.heard_through||{}) do |heard_through_fields| %>
  <% User::HEARD_THROUGH_VALUES.each do |heard_through_val| %>
    <%= heard_through_fields.check_box "field %>
    <%= heard_through_fields.label :heard_through,heard_through_val %>
  <% end %>
<% end %>
```

如同 Boolean 的版本，這個方法一樣不用新增 Model，也減低了程式的複雜度、也不用多出這麼多的 Boolean 欄位，但還是有缺點，例如：使用 `seriailize` 的同時其實也失去了 Rails 本身提供的一些欄位的方法像是 `find`，你必須自己去些一些字串比對的方法來幫助你建立出 find 的功能，因此如果這些資料的特性是開放式的( open-ended )，例如說由用戶自行產生的複雜資料，從別的程式撈出來的資料，夾帶著無法預知的參數，這種情況我們也不太會需要 find 方法來搜尋這些資料，因此就是一個使用 `serialize` 方法的好時機。

除了以上的方法，你也可以考慮將這些資料以序列化的屬性儲存在父類別，例如[Act as Revisionable](https://github.com/bdurand/acts_as_revisionable) 這支 Gem，可以將你任何的 Model 都做到版本化的功能，這支 Gem 會在你每個需要版本的 Model 都建立一個新的同樣的 Table 用來以序列化的形式(並不是由 Rails 本身提供的 serialize 方法)儲存你的版本，例如你有一個 Document 的 Model，那麼就會建立一個 doecument_versions 的 Table 用來儲存這些版本資料，這樣的方式讓一個 Table 可以忽略 Model 本身的屬性和 Association 來儲存很多版本 Model。

還有一個例子是像是 Thoughtbot 的 Hoptoad ( Airbrake的前身 )，是一個可以幫助你追蹤你的程式產生 Exception 的應用程式， Hoptoad 以序列化的方式儲存每個你程式所產生的 Exception，還包含了 request 的類型、session data、和伺服器環境等資訊，全部經過序列化後儲存起來，也由於是儲存序列化後的資料的關係，Hoptoad 不僅支援 Rails，只要你的程式可以被格式化成 Hoptoad 所支援的格式就可以，也因此這樣的技巧大幅的減低你的程式複雜度，取而代之使用序列化的資訊將複雜的資料儲存起來，這正是我們想要的。

當然因為 Rails 是 Hoptoad 主要支援的平台，而且你也希望能在 Hoptpad 的介面上看到各種 Exception 產生時的重要資料，例如 Rails 的環境、哪支 Controller 的 Action 發生的、哪一支檔案的第幾行所產生的 Exception 等，這些資訊都會在 Hoptoad 的 Notice Model 中被記錄：

``` ruby
before_validation :extract_backtrace_info, :on => :create
before_validation :extract_request_info, :on => :create
before_validation :extract_environment_info, :on => :create
private
  def extract_backtrace_info
    unless backtrace.blank?
      self.file, self.line_number = backtrace.first.split(':')
    end
  end
  def extract_request_info
    unless request.blank? or request[:params].nil?
      self.controller = request[:params][:controller]
      self.action = request[:params][:action]
    end
  end
  def extract_environment_info
    unless environment.blank?
      self.rails_env = environment['RAILS_ENV']
    end
  end
```

前面說過，當你選擇使用 serialize 的方式記錄資料時，你便失去了方便使用 find 方法的功能，但像是 reversion 或是 Hoptoad 這樣的情況，尋找資訊對我們來說通常是不必要的，因此我們可以大膽放心的使用這樣的方式來取代大量的新增 Model。

##心得：
這個章節最主要的重點就是希望當你在思考著新增一個 Model 時，或許可以經由他所提供的檢查方法來考慮一下自己是否真的需要一個 Model 來實作，以一般小專案而言其實這樣的方式也沒什麼錯誤，但其實乖乖的用正規化也沒什麼不對，以上面的 Role 的例子來說，多對多的關聯在想法上也是完全正確的，或許新增一個 Model 的確會增加一些成本，但其實在大型專案中我們會要求盡量作正規化，除非需求真的太小。

另外就是過度預期的問題，這個問題的解法說簡單也簡單，就是 TDD 就是了，在 TDD 的開發模式中我們都會先寫測試，這樣的方式會讓我們在寫真的 production 的 code 時不會超出測試的範圍，而且也能在我們寫測試的同時以行為面去思考，因此在這樣的開發方式下不太會發生這樣的情況，雖然學寫測試不是一件容易的事，但學會寫測試看到一個一個的綠燈心情還是很愉快的阿！
