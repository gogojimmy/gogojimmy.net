---
layout: post
title: "在Rails中使用Path或是Url的時機"
date: 2011-12-02 23:17
comments: true
categories: Rails
---
剛開始寫Rails的時候，一個很基本的問題就是Path跟Url的分別，其實差別在於一個回傳的是相對路徑，而一個是絕對路徑而已，例如說假設今天我們有個叫做Product的Resource

``` ruby routes.rb
resources :products
```

當你使用path的時候，像是這樣

``` ruby
link_to "產品首頁", products_path
```

這段程式碼會產生：

``` html
<a href="/products">產品首頁<a>
```

而使用url的時候，像是這樣

``` ruby
link_to "產品首頁", products_url
```

這段程式碼會產生：

``` html
<a href="http://www.example.com/products">產品首頁<a>
```

那什麼時候該用url,什麼時候又該用path呢?

一般來說在View的時候我們會用path，而在Controller的時候我們會用url，因為在View的時候使用path連到我們自己domain底下的相對網址會比較節省時間也提昇些許效能，而url通常我們會使用在redirect_to的3xx轉向，這種情況就得使用絕對路徑因此我們會使用url，還有種情況你必須使用url，就是在有SSL的網址與沒有SSL的網址間互傳的時候，你必須使用絕對路徑，因此請使用url。

不過為了一些SEO的理由，使用絕對路徑會比使用相對路徑來的優秀，例如說root_url會比root_path來的好。
