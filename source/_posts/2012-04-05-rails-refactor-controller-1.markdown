---
layout: post
title: "[Rails 重構]Controller(1)-拋開複雜的邏輯"
date: 2012-04-05 20:52
comments: true
categories: Rails, Refactor, Controller, AntiPatterns
published: false
---
Controller 是我們在使用 Rails 的 MVC 架構時，所有動作的起點，所有的 request 都會先到 Controller 中，經過處理後最後再傳送給 View 做顯示，這樣的設計原則讓我們通常在寫 Rails 的程式時都從 Controller 先開始寫，因此也容易不知不覺的將一些複雜的東西放到 Controller 中處理，我們在前面的部份已經學過了要將複雜的邏輯放到其他地方處理，像是 Model 和 Helper，Controller 要保持越簡單越好，這樣子一個很大的原因是在 Controller 保持簡單的情況下，我們可以把邏輯拆成數個簡單的方法在其他地方來實作，這樣一來我們就可以針對這些方法來寫單元測試，以驗證每個小功能都能運作無誤，如果今天把這些複雜的邏輯一股腦的都放在 Controller 中串接起來，今天中間有一個邏輯的部份失效你很難知道問題在哪，也很難對這樣的 Controller 做有彈性的更動，因此這篇就要來談談 Controller 的寫法。

<!-- more -->

##問題：手工 Controller

大多的情況在 Controller 中處理的都是 Model 的 CRUD，但我們也常遇到比較複雜的需求，例如使用者註冊、登入、認證等流程，這些動作包含了很多的檢查機制，意味著會有很多邏輯，當這樣的東西一多就很難保持 Controller 的乾淨，在以往使用 plugin 的情況也還是很容易讓你的 Code 變得複雜難以理解，因此我們可能還是會自己去打造自己想要的部份，但做這樣重複造輪子的動作也是違反了我們 DRY 的原則，因此這裡我會推薦使用人家寫好的 gem 來讓你省功夫。

* [Clearance](https://github.com/thoughtbot/clearance)

  [Clearance](https://github.com/thoughtbot/clearance) 是由 [Thoughtbot](http://thoughtbot.com/) 這家公司推出的 gem，可以方便的幫你掛上 Authentication with Email & Password
* [Authlogic](https://github.com/binarylogic/authlogic)

  [Authlogic](https://github.com/binarylogic/authlogic) 幫你加上一個認證的 Model，幫你省去重新打造認證機制的時間。

##問題：笨重的 Controller

###重構：使用 ActiveRecord 的 Callback 與 Setter

來看一段從真實 production 的程式中擷取出來的程式：

``` ruby
class ArticlesController < ApplicationController
  def create
    @article = Article.new(params[:article])
    @article.reporter_id = current_user.id
    begin
      Article.transaction do
        @version = @article.create_version!(params[:version], current_user)
      end
    rescue ActiveRecord::RecordNotSaved, ActiveRecord::RecordInvalid
      render :action => :index and return false
    end
    redirect_to article_path(@article)
  end
end
```

這個 create action 中有幾個問題，在我們重構這些問題前有件事情是你必須在動手前先思考的，如果今天這不是你的程式，那麼你最好先去看他的單元測試的部份，因為測試有助於你進一步了解程式的用途，可能他為了一些理由不得不寫出這樣的程式，那麼你可以在他的測試中看出端倪，防止你採到前面那個人採到的地雷。

讓我們回頭看看這支程式，我們先看到 `Article.transaction` 這一塊 block，這個區塊是做資料庫的 transaction，通常是為了一個需要確定所有動作都能確實執行的程序，若有一個環節出了錯誤則將全部動作 rollback，第一個問題就是 database 的 transaction 不應該放在 Controller 中實作，因為 Controller 不該做這麼多的事情，尤其是一群互相依賴的程式，第二是其實 Active Record 生命週期裡的方法通常已經被包成 transaction 了，例如說你看到 `@article.create_version!(params[:version], current_user)` 裡並沒有包含 `save` 這個方法。

因此我們的想法是，既然像是 `save` 這個方法已經被包成 transaction 了，那麼我們可以簡單的使用 `save` 來取代複雜的 transaction 區塊，但要如何去確定完成這麼多的動作呢？答案就是使用 model 本身的 Callback 與 setter，在這個例子中就是指 Article 與 Version。

最後在程式中有這個部分 `rescue ActiveRecord::RecordNotSaved, ActiveRecord::RecordInvalid`，通常我們不會在 Controller 的流程中放入 exceptions，因為在 web application 中使用者看到 exception 不會知道該怎麼處理，這應該是由程式自動產生好的事情，我們應該簡單的使用 GOTO 來取代，基於剛剛的想法，我們的程式重構後可能會長這樣：

``` ruby
class ArticlesController < ApplicationController
  def create
    @article = Article.new(params[:article])
    @article.reporter_id = current_user.id
    if @article.save
      redirect_to article_path(@article)
    else
      render :action => :new
    end
  end
```

在新的程式中我們想以簡單的 `save` 方法來取代原本複雜的 transaction，並且當儲存不成功而 `save` 回傳 false 的時候將使用者引回 `new` action 重新輸入，因此我們將以這支程式作為我們重構的目標，因此第一步我們先看到原先的 `create_version!` 方法：

``` ruby
def create_version!(attributes, user)
  if self.versions.empty?
    return create_first_version!(attributes, user)
  end
  # mark old related links as not current
  if self.current_version.relateds.any?
    self.current_version.relateds.each do |rel|
      rel.update_attribute(:current, false)
    end
  end
  version = self.versions.build(attributes)
  version.article_id = self.id
  version.written_at = Time.now
  version.writer_id = user.id
  version.version = self.current_verison.version + 1
  self.save!
  self.update_attribute(:current_version_id, version.id)
  version
end
```

在這方法裡第一件做的事情就是當這個 article 還沒有任何版本時，我們呼叫 `create_first_version!` 方法來建立第一個版本，那我們再來看看這個方法：

``` ruby
def create_first_version!(attributes, user)
  version = self.versions.build(attributes)
  version.written_at = Time.now
  version.writer_id = user.id
  version.state ||= "Raw"
  version.version = 1
  self.save!
  self.update_attribute(:current_version_id, version.id)
  version
end
```

你會發現的第一個問題就是與前一個 `create_version!` 方法相比，有很多重複的程式碼，這將是我們重構的第一步：

####使用 Rails 內建的欄位
回頭看看前面兩個方法中都有個 `version.written_at`，用來記錄 version 的產生時間，其實 Rails 本身就有內建 `created_at` 的欄位，當這筆記錄建立時便會自動將當時時間寫入資料庫，因此我們根本不用自己建立這個部分。

####將預設值寫在資料庫的 Migration 中
在 `create_first_version` 方法中有 `version.state ||= "Raw"` 這個部分用來建立版本的狀態，當使用者沒有提供時我們就預設為 "Raw"，這部份我們可以寫在 Migration 的檔案中建立資料庫欄位的預設值：

``` ruby
class AddRawDefaultToState < ActiveRecord::Migration
  def self.up
    change_column_default :article_versions, :state, "Raw"
  end
  def self.down
    change_column_default :article_versions, :state, nil
  end
end
```

但是必須注意的是，這樣的寫法是不會經過 validation 的，愛注意。

####Callback
回頭看到我們的 `create_verstion` 與 `create_first_version` 兩個方法，其中各有一個部分是這樣寫的 `version.version = self.current_verison.version + 1`、`version.version = 1`，這兩個動作都是用來為版本建立版本編號，最好的方法其實我們可以使用 Callback 機制來取代，因此我們可以將他寫在 Version Model 中：

``` ruby
class Version < ActiveRecord::Base
  before_validation :set_version_number, :on => :create
  validates :version, :presence => true

  private
  def set_version_number
    self.version = (article.current_version ?  article.current_version.version : 0) + 1
  end
end
```

如此一來當每個版本建立前就會自動的增加版本號了。

####找出沒用的部分

經過剛剛的重構後，我們回頭看看現在兩個方法：

``` ruby
def create_version!(attributes, user)
  if self.versions.empty?
    return create_first_version!(attributes, user)
  end
  # mark old related links as not current
  if self.current_version.relateds.any?
    self.current_version.relateds.each |rel| do
      rel.update_attribute(:current, false)
    end
  end
  version = self.versions.build(attributes)
  version.article_id = self.id
  version.writer_id = user.id
  self.save!
  self.update_attribute(:current_version_id, version.id)
  version
end

def create_first_version!(attributes, user)
  version = self.versions.build(attributes)
  version.writer_id = user.id
  self.save!
  self.update_attribute(:current_version_id, version.id)
  version
end
```

你可以發現對 `create_first_version!` 這個方法來說，與前一個方法比較起來只差了一行 `version.article_id = self.id`，而這行程式其實是一點都不需要的，因為在呼叫上一行的 `versions.build` 時已經會自動執行了，因此我們完全可以移除這一行不必要的程式。

####另一個 Callback

現在兩個方法不一樣的部份只剩下這個部分：

```
# mark old related links as not current
if self.current_version.relateds.any?
  self.current_version.relateds.each |rel| do
    rel.update_attribute(:current, false)
  end
end
```

這個部分的用途在於當你不是建立第一個版本的時候，那麼將原先連結為現在版本的的連結全部移除，也就是不再是最新版本的意思，所以你可以用一個條件來判斷現在是否建立的是新版本就好，因此你現在可以完全把 `create_first_version!` 移除掉，經過改寫後我們現在的 `create_version!` 看起來會是這樣：

``` ruby
def create_version!(attributes, user)
  unless self.versions.empty?
    # mark old related links as not current
    if self.current_version.relateds.any?
      self.current_version.relateds.each |rel| do
        rel.update_attribute(:current, false)
      end
    end
  end
  version = self.versions.build(attributes)
  version.writer_id = user.id
  self.save!
  self.update_attribute(:current_version_id, version.id)
  version
end
```

然後我們甚至可以把 unless 區塊搬到 Version Model 中作為另一個 Callback：

``` ruby
class Version < ActiveRecord::Base
  before_validation_on_create :set_version_number
  before_create :mark_related_links_not_current
  private
    def set_version_number
      self.version = (article.current_version ?  article.current_version.version : 0) + 1
    end
    def mark_related_links_not_current
      unless article.versions.empty?
        if article.current_version.relateds.any?
          article.current_version.relateds.each |rel| do
            rel.update_attribute(:current, false)
          end
        end
      end
    end
```

然後你會發現其實 `if article.current_version.relateds.any?` 這一行也是不必要的，因為如果沒有任何元素的話其實下面的區塊就也只是回傳空陣列，而非 nil，因此我們可以把那一段程式也拿掉。現在我們把目光放到 `unless` 的區塊，基本上我不喜歡寫 `unless` 是因為他沒有 `if` 來的直覺，很多時候容易讓人判斷錯誤，因此我會喜歡把它改寫成 `if` 的判斷讓人比較好理解，因此他會變成 `if article.current_version` 這樣的區塊：

``` ruby
def mark_related_links_not_current
  if article.current_version
    article.current_version.relateds.each do |rel|
      rel.update_attribute(:current, false)
    end
  end
end
```
你會發現在這個方法裡就呼叫了兩次的 `article.current_version`，回想一下 Law of Demeter，我們應該要將 `current_version` 封裝在 Version 中，因此我們再一次的改寫這個 Model：

``` ruby
class Version < ActiveRecord::Base
  before_validation :set_version_number, :on => :create
  before_create :mark_related_links_not_current, :if => :current_version
  private
    def current_version
      article.current_version
    end
    def set_version_number
      self.version = (current_version ? current_version.version : 0) + 1
    end
    def mark_related_links_not_current
      current_version.relateds.each do |rel|
        rel.update_attribute(:current, false)
      end
    end
end
```

注意到我們將 `current_version` 封裝後，也把 `if` 判斷自方法中抽離，直接在 `before_create` 的 Callback 中做判斷即可，這樣一來程式就會顯得更直觀，在方法中越少 if-else 的判斷程式就越容易理解。

####再來一個 Callback
我們剛剛已經完成了一些重構，我們將一些邏輯都搬到 Version Model 中的 Callback 中去實作，我們現在回頭來看看我們的 `create_version!` 方法：

``` ruby
def create_version!(attributes, user)
  version = self.versions.build(attributes)
  version.writer_id = user.id
  self.save!
  self.update_attribute(:current_version_id, version.id)
  version
end
```

注意到上面當每次新版本建立成功後，我們使用 `update_attribute` 來將目前的版本更新為剛剛建立的版本，我們也可以把它搬到 Callback 中實作：

``` ruby
class Version < ActiveRecord::Base
  before_validation :set_version_number, :on => :create
  before_create :mark_related_links_not_current, :if => :current_version
  after_create :set_current_version_on_article
  private
  def set_current_version_on_article
    article.update_attribute :current_version_id, self.id
  end
end
```

再來回頭看看我們的 `create_version!`：

``` ruby
def create_version!(attributes, user)
  version = self.versions.build(attributes)
  version.writer_id = user.id
  self.save!
  version
end
```

你會發現除了 `save!` 方法之外只剩下兩行的敘述，回想我們的目標，我們希望在 Controller 中只剩下一個 `save` 方法來取代原先的 transaction，我們已經將大部分都搬到 Callback 中實作了，因此我們可以把剩下的兩行搬回 Controller 中，這樣 Controller 就可以簡單的來呼叫 `save!` 來完成剛剛的 transaction 了：

``` ruby
class ArticlesController < ApplicationController
  def create
    @article = Article.new(params[:article])
    @article.reporter = current_user
    @version = @article.versions.build(params[:version])
    @version.writer = current_user
    if @article.save
      render :action => :index
    else
      redirect_to article_path(@article)
    end
  end
end
```

比較原本的程式，首先我們移除了 transaction 的區塊，因為 `save` 方法本身就實作了 transaction 的機制，另外我們使用沒有驚嘆號版本的 `save` 來取代原本在 `create_version` 方法中的 `save!`，這麼做的目的是因為在這邊的資料是由使用者輸入的，使用 `save!` 當錯誤發生時會拋出例外而你必須去做例外處理，那不如就直接使用 `save` 方法使用簡單的 true-false 來取代例外處理，所以我們連帶的把例外處理的部份一起拿掉了。 另外我們還使用 `@article.reporter = current_user` 來取代原先的 `@article.reporter_id = current_user.id`，這樣能防止我們出錯使用錯誤的東西指派到 id 欄位。

現在的 Controller 已經長得很像我們當初預想的狀況了，但還是有些東西可以改善，我們看到 `@version` 仰賴由前端傳來的 `params[:version]` 參數來建立一個新版本，我們可以把這個動作丟到 Article Model 中讓他來負責建立一個新版本，我們可以建立一個叫做 `new_version` 的 setter 和 getter，這樣做的好處是在於這樣我們就可以在前端的 form 裡面巢狀包進這個 version 欄位，傳回來的參數就是 `params[:article][:version]`，這樣一來當呼叫 `Article.new` 的時候就會直接的將 version 的 hash 傳給 `new_version` 這個 setter：

``` ruby
class Article < ActiveRecord::Base
  def new_version=(version_attributes)
    @new_version = versions.build(version_attributes)
  end
  def new_version
    @new_version
  end
end
```

有了新的 getter 與 setter ，你最後的 Controller 看起來就會是這樣：

``` ruby
class ArticlesController < ApplicationController
  def create
    @article = Article.new(params[:article])
    @article.reporter = current_user
    @article.new_version.writer = current_user
    if @article.save
      render :action => :index
    else
      redirect_to article_path(@article)
    end
  end
end
```

我們這樣使用 Callback 以及 setter 來重構除了讓 Controller 瘦身之外，這些 Callback 與 setter 也能重複利用，但是不當過度使用 Callback 也可能讓程式產生一些難以找出的 Bug，同樣的道理也適用在 setter 身上，容易變得 "矯枉過正"。這時候，Presenter 模式將也是一個答案。

###Presenter Pattern

在上一個重構中我們提到了使用 Callback 與 setter 來將複雜的邏輯層搬到主要的 Model 中去實作，來保持 Controller 的乾淨，但是在現實中我們常遇到更複雜的情況是 Callback 與 setter 都不夠用的，例如說我們必須要一個 create 動作中實作出好幾個不同的 model，例如說下面這個測試：

``` ruby
class AccountsControllerTest < ActionController::TestCase
  context "on GET to #new" do
    setup { get :new }
    should assign_to(:account)
    should assign_to(:user)
    should render_template(:new)
    should "render form for account and user" do
      assert_select "form[action$=?]", accounts_path do
        assert_select "input[name=?]", "account[subdomain]"
        assert_select "input[name=?]", "user[email]"
        assert_select "input[name=?]", "user[password]"
      end
    end
  end
  context "on POST to #create with good values" do
    setup do
      post :create,
        :account => {:subdomain => "foo"},
        :user => {:email => "foo@bar.com",
                  :password => "issekrit?"}
        end
    should set_the_flash.to(/created/i)
    should_change "User.count", :by => 1
    should_change "Account.count", :by => 1
    should "assign the user to the account" do
      assert_equal assigns(:account).id, assigns(:user).account_id
    end
  end
  context "on POST to #create with bad account values" do
    setup do
      post :create,
        :account => { },
        :user => {:email => "foo@bar.com", :password => "issekrit?"}
    end
    should assign_to(:account)
    should assign_to(:user)
    should render_template(:new)
  end
  context "on POST to #create with bad user values" do
    setup do
      post :create,
        :account => { :subdomain => "foo" },
        :user => { }
    end
    should assign_to(:account)
    should assign_to(:user)
    should render_template(:new)
  end
end
```

這個測試敘述了使用者建立一個帳號時需建立一個含有 subdomain 的 account 以及一個有 email 和 password 的 user，且必須同時儲存建立否則兩個都應該 rollback，在 Controller 的實作大概會是這樣：

``` ruby
class AccountsController < ApplicationController
  def new
    @account = Account.new
    @user = User.new
  end
  def create
    @account = Account.new(params[:account])
    @user = User.new(params[:user])
    @user.account = @account
    if @account.save and @user.save
      flash[:notice] = 'Account was successfully created.'
      redirect_to(@account)
    else
      render :action => "new"
    end
  end
end
```

而在 View 中我們使用 `form_for` 中包 `fields_for` 來同時建立 account 與 user：

``` ruby
<h1>New account</h1>
<%= form_for(@account) do |f| %>
  <%= f.error_messages %>
  <p>
  <%= f.label :subdomain %><br />
  <%= f.text_field :subdomain %>
  </p>
  <%= fields_for(@user) do |u| %>
    <%= u.error_messages %>
    <p>
    <%= u.label :email %><br />
    <%= u.text_field :email %>
    </p>
    <p>
    <%= u.label :password %><br />
    <%= u.text_field :password %>
    </p>
  <% end %>
  <p>
  <%= f.submit "Create" %>
  </p>
<% end %>
```

這樣的寫法會錯誤，因為在 Controller 中我們的 create 方法是這樣寫的：

``` ruby
  if @account.save and @user.save
```

發現了嗎？我們先判斷 `@account.save` 成功後再去判斷 `@user.save`，所以使用者要是傳了一個正確的 account 與不正確的 user，那麼就會建立一個新的 account 但是沒有 user，因此在測試時就會錯掉，這個問題我們只能從資料庫的 transaction 來下手，因此我們改寫 Controller：

``` ruby
class AccountsController < ApplicationController
  def new
    @account = Account.new
    @user = User.new
  end
  def create
    @account = Account.new(params[:account])
    @user = User.new(params[:user])
    @user.account = @account
    ActiveRecord::Base.transaction do
      @account.save!
      @user.save!
    end
    flash[:notice] = 'Account was successfully created.'
    redirect_to(@account)
  rescue ActiveRecord::RecordInvalid, ActiveRecord::RecordNotSaved
    render :action => "new"
  end
end
```

我們使用 `ActiveRecord::Base.transaction` 區塊來包覆著兩個 `save!` 方法，確保其中一個失敗時都會 rollback，同樣的我們必須使用 `save!` 來確保例外的拋出讓我們能去處理 transaction 的例外，但是我要告訴你到目前為止這完全是一個錯誤的思考方向，基於之前說過的我們不應該在 Controller 中處理複雜的例外機制與 transaction，因此這個問題已經不是 RESTful controller 能處理的範圍，我們需要新的東西，那就是 MVP(Model-View-Presenter)，MVP 模式主要就是應付在處理 Controller 中處理不來的複雜邏輯問題，簡單來說就是用一個新的 Presenter Model 再去把複雜的部份包起來做，一個概念會是這樣：

``` ruby
class SignupTest < ActiveSupport::TestCase
  should validate_presence_of :account_subdomain
  should validate_presence_of :user_email
  should validate_presence_of :user_password
  should "be a presenter for account and user" do
    assert_contains Signup.new.presented.keys, :account
    assert_contains Signup.new.presented.keys, :user
  end
  should "assign the user to the account on save" do
    signup = Signup.new(:account_subdomain => "subdomain",
                        :user_email => "e@mail.com",
                        :user_password => "passw0rd")
    assert signup.save
    assert user = signup.user
    assert account = signup.account
    assert_equal account.id, user.account_id
  end
end
```

我們希望這個叫做 Signup 的 presenter 能幫我處理那些邏輯的部份，在第一個 should 中確保 Signup Class 是一個負責 user 與 account 的 Active Presenter Class，你不會希望讓 user 與 account 重複測試(Presenter 自己本身就能測試了)，在下面則是確保 user 及 account 的 save 都能成功，這樣的 Signup class 會長這樣：

``` ruby
class Signup < ActivePresenter::Base
  before_save :assign_user_to_account
  presents :user, :account
  private
    def assign_user_to_account
      user.account = account
    end
end
```

這個 Signup 繼承於 `ActivePresenter::Base`，因此會負責確認所有的 Callback 是否成功，這樣一來你就擁有一個看起來像是單一 ActiveRecored 的 Model 了，例如說你的 View 就會變成這樣：

``` ruby
<h1>Signup!</h1>
<%= form_for(@signup) do |f| %>
  <%= f.error_messages %>
  <p>
  <%= f.label :account_subdomain %><br />
  <%= f.text_field :account_subdomain %>
  </p>
  <p>
  <%= f.label :user_email %><br />
  <%= f.text_field :user_email %>
  </p>
  <p>
  <%= f.label :user_password %><br />
  <%= f.text_field :user_password %>
  </p>
  <p>
  <%= f.submit "Create" %>
  </p>
<% end %>
```

這樣一來，你回頭去改原先針對 Controller 的 test，將原先的 fields 名稱改為新的像是 `account_subdomain`，這樣一來你的測試就會過了，現在的 Signup Controller 就會是一個看起來很簡單的情況：

``` ruby
class SignupsController < ApplicationController
  def new
    @signup = Signup.new
  end
  def create
    @signup = Signup.new(params[:signup])
    if @signup.save
      flash[:notice] = 'Thank you for signing up!'
      redirect_to root_url
    else
      render :action => "new"
    end
  end
end
```
