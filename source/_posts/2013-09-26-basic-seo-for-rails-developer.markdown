---
layout: post
title: "給 Rails Developer 的基本SEO"
date: 2013-09-26 17:03
comments: true
categories: ['SEO', 'Rails', 'Meta Tags']
---

做過那麼多的Projects幾乎每個老闆都會要求SEO，幾個專案下來我也整理了一些SEO基本佈置，在這邊就分享一下，這裡並沒有太高深的SEO技巧，都是基本SEO所需要的配置，但因為是基本，所以非常重要，如果你平常寫筆記有在重點部分畫星星的習慣，請把這篇印下來在上面畫上一個銀河。

<!-- more -->

## 搞定Meta Tags 及 Meta Data

第一步也是最基本的一步也就是搞定在`head`裡面的那些Meta Tag以及一些給特殊網站使用的Meta Data，下面是你會需要加入的東西：

* Title Tag：每一頁的標題，英文長度最好是70個字內，中文大約40字內，關鍵字盡量往前放，每一頁的`Title Tag`不要重複。
* Meta Description Tag：每一頁的敘述，包含標點符號在內英文大約150字內，中文12字內，關鍵字最好在前10個字就出現，會出現在Google搜尋結果，因此文案好壞普遍來說會影響點擊率，每一頁的`Decription Tag`也盡量不要重複。
* Canonical Url：告訴搜尋引擎關於這個頁面的標準網址，有時候我們一個網頁會因為參數而產生不同的網址，這標籤能告訴搜尋引擎只需要收錄哪個網址即可。
* Facebook Open Graph：專門給Facebook抓的，可以用來調整你的文章分享到Facebook後的的縮圖、標題、敘述，搭配其他或是自定義的Open Graph可以做更多應用。
* Twitter Cards：專門給Twitter抓的，可以用來調整你的文章分享到Twitter後的樣式。

以上這些`Meta Tag`我都使用[meta-tag](https://github.com/kpumuk/meta-tags)來完成，例如：

``` ruby Title Tag
set_meta_tags title: '給Rails Developer的基本SEO'
# <title>給Rails Developer的基本SEO</title>
set_meta_tags site: '好麻煩部落格', title: '給Rails Developer的基本SEO'
# <title>好麻煩部落格 | 給Rails Developer的基本SEO</title>
set_meta_tags site: => '好麻煩部落格', title: '給Rails Developer的基本SEO', reverse: true
# <title>給Rails Developer的基本SEO | 好麻煩部落格</title> <= 建議作法
```

``` ruby Description Tag
set_meta_tags description: '教你怎麼在Rails裡做好基本的SEO'
# <meta name="description" content="教你怎麼在Rails裡做好基本的SEO" />
```

* [更多關於Meta Tag](https://support.google.com/webmasters/answer/35624?hl=zh-Hant&ref_topic=2371375)

``` ruby Canonical Url
set_meta_tags canonical: 'http://gogojimmy.net'
# <link rel="canonical" href="http://gogojimmy.net" />
```

* [更多關於Canonical Url](https://support.google.com/webmasters/answer/139394?hl=zh-Hant&ref_topic=2371375)

``` ruby Facebook Open Graph
set_meta_tags og: {
  title: '教你怎麼在Rails裡做好基本的SEO',
  site_name: '好麻煩部落格'
  type: 'website',
  url: 'http://gogojimmy.net',
  image: 'https://dl.dropboxusercontent.com/u/1390253/gogojimmy.jpg',
}
# <meta property="og:title" content="教你怎麼在Rails裡做好基本的SEO"/>
# <meta property="og:type" content="website"/>
# <meta property="og:site_name" content="好麻煩部落格"/>
# <meta property="og:url" content="http://gogojimmy.net"/>
# <meta property="og:image" content="https://dl.dropboxusercontent.com/u/1390253/gogojimmy.jpg"/>
```

Facebook Open Graph可以做出很多應用，例如你可以申請一個Facebook APP之後自己定義新的動作來出現在Facebook上，國內像是[KKBOX](http://kkbox.com)，[icook 愛料理](http://icook.tw)都有使用。

* [更多關於Facebook Open Graph](http://developers.facebook.com/docs/opengraph/)

``` ruby Twitter Card
set_meta_tags twitter: {
  card: => "summary",
  site: => "@gogojimmy"
}
# <meta property="twitter:card" content="summary"/>
# <meta property="twitter:site" content="@gogojimmy"/>
```

Twitter Card目前支援七種Card，像是`Summary`、`Photo`、`Gallery`、`Product`之類的基本上如果你有設定Facebook Open Graph的話，你可以不用設定Twitter Card，因為Twitter抓不到Twitter Card的時候會尋找Open Graph的標籤。

* [更多關於Twitter Card](https://dev.twitter.com/docs/cards/)

另外還有Google+ Authorship以及Google+ Publisher、前者是給一般個人使用、後者是企業公司品牌使用，這樣搜尋引擎出現的內容會出現你的頭像以及讓Google知道這篇文章的正確作者是誰：

``` ruby Google+ Authorship
link_to 'Google', "#{你的google_profile_url}?rel=author"
# <a href="你的google_profile_url?rel=author">Google</a>
```

``` ruby Google+ Publisher
link_to 'Google', "#{你的google_page_profile_url}?rel=publisher"
# <a href="你的google_page_profile_url?rel=author">Google</a>
```

注意以上兩者皆需要在你的個人/官方的Google+ Profile裡面新增你的網站才會生效。

* [更多關於Google+ Authorship](https://support.google.com/webmasters/answer/2539557)
* [更多關於Google+ Publisher](https://support.google.com/webmasters/answer/1708844)

## 搞定連結

連結大概分三種，純文字的連結、圖片連結以及你不想讓搜尋引擎作為SEO計算的nofollow連結(像是留言裡的垃圾回應或是付費連結)

``` ruby 純文字連結
link_to '關鍵字', some_path
# 錨點文字就是關鍵字
```

``` ruby 圖片連結
link_to some_path do
  image_tag('關鍵字.jpg', alt: '關鍵字')
end
# 對圖片連結來說，alt的用途就是錨點文字，檔案名稱也要符合關鍵字。
# 沒意義的圖片不要做SEO的強化，像是網站內自己的素材。
# 圖片附近最好有描述圖片的上下文，因為這會出現在搜尋結果中(請針對自己網站的設計做調整)。
# 如果圖片有多種尺寸，利用Robots.txt等方式讓搜尋引擎只收錄一個尺寸，簡單來說就是圖片不要重複。
```

``` ruby nofollow連結
link_to '某個不想被follow的地方', some_path, rel: 'nofollow'
# nofollow會讓搜尋引擎不會前往這個連結，通常運用在留言、付費連結，
# 以及像是登入登出這種沒有必要讓搜尋引擎收錄的地方。
```
* [更多關於圖片連結](https://support.google.com/webmasters/answer/114016?hl=zh-Hant&ref_topic=2370565)
* [更多關於nofollow](https://support.google.com/webmasters/answer/96569?hl=zh-Hant)

## 搞定網址

良好的網址對於SEO非常重要，一般在Rails裡面使用`[:id]`的方法我們可以使用[friendly_id](https://github.com/norman/friendly_id)這套Gem來幫助我們，你可以把像是`http://example.com/products/123`變成`http://example.com/products/好商品不買嗎`這樣的網址，除此之外他最好用的功能就是可以記錄你的網址記錄，例如說你原先的網址已經被搜尋引擎收錄，但今天你有需求要把網址改掉，這時候搜尋引擎點進來的人就會404找不到網頁，但是[friendly_id](https://github.com/norman/friendly_id)可以幫我們把原先的網址用301轉址到新的網址，對SEO有非常重要的幫助，[friendly_id](https://github.com/norman/friendly_id)使用上很簡單，你看完他的[ReadMe](https://github.com/norman/friendly_id)就沒問題了。

其他關於URL的建議：

* 網址內要有關鍵字，越前面越好。
* 注意因為不同參數所產生同樣的網頁，使用Canonical Url的方式來避免重複內容，除非你是搜尋結果頁。
* 將內容放在網域下第一層路徑的效果比弄成子網域好，因為可以增加網域的權威性，例如`http://example.com/blog`會比`http://blog.example.com`來的好，因為你的內容可以增加`http://example.com`的權威性。

## 搞定爬蟲

robots.txt這隻檔案是用來告訴搜尋引擎什麼東西不要收錄，在Rails中預設是放在`public/robots.txt`，你可以直接修改這隻檔案，或是說如果你有多個測試環境(像是staging)，那麼你大概不想讓Google可以收錄staging的內容，因此這時候你需要可以針對Rails環境來動態產生的`Robots.txt`，我是參考[這支Gist的作法](https://gist.github.com/timcheadle/3761844)。

* [更多關於Robot.txt](http://www.robotstxt.org/robotstxt.html)

## 搞定Sitemap

Sitemap可以直接告訴搜尋引擎你的網站結構，幫助搜尋引擎更容易了解你的網站內容，通常我的習慣是會在每次有新內容產生的時候就會向搜尋引擎提交一個新的Sitemap，我是使用[sitemap_generator](https://github.com/kjvarga/sitemap_generator)這套Gem來實作的，這個Gem在使用上也很簡單，把[ReadMe](https://github.com/kjvarga/sitemap_generator)看一看就行了，他有提供`rake task`讓你可以在新內容產生的時候跑`rake`去提交新的sitemap到所有的搜尋引擎。

更多關於Sitemap的說明與建議：

* [Sitemap的寫法](https://support.google.com/webmasters/answer/183668?hl=zh-Hant&ref_topic=8476)
* 可以針對不同的Content Type來做不同的Sitemap，像是影片、圖片、手機版，[sitemap_generator](https://github.com/kjvarga/sitemap_generator)這套Gem你可以輕鬆做好。

## 搞定語系與國家

別忘了在你的html tag中加入語系

``` html
<html lang="zh">
```

你可以在head tag中加入所有語系以及國家，可以使用[meta-tag](https://github.com/kpumuk/meta-tags)來幫助你完成

``` html
<link rel="alternate" hreflang="x-default" href="http://www.example.com/" /> (提供自動導向或是讓使用者可以自己選擇語言或地區的)
<link rel="alternate" hreflang="en" href="http://example.com/en/" /> (英文版本的入口)
<link rel="alternate" hreflang="en-ca" href="http://example.com/en-ca/" /> (針對加拿大地區使用的英文版本)
```

你也可以使用Sitemap來產生語系，請參考[Sitemap的寫法](https://support.google.com/webmasters/answer/183668?hl=zh-Hant&ref_topic=8476)，基本上面那種方法（`<link rel="alternate" hreflang=""`）與Sitemap的作法擇一就可以了。

## 其他建議

可以的話，請盡量提供行動裝置的網站，無論是Responsive Design或是建立另外一個網站，Google在他們的[Building Mobile-Optimized Websites](https://developers.google.com/webmasters/smartphone-sites/)裡面有提到Responsive對Google來說最方便，因為這樣子對他們來說內容都一樣，但是並不是說Responsive就是你唯一的標準，如果你的網站內容很複雜的話，建議還是拆成不同的網頁來呈現，無論是不是同個網址。

* 參考[網站優化應該使用Responsive Web Design嗎?](http://seo.dns.com.tw/?p=10633)

有什麼錯誤都歡迎來信或留言罵我，謝謝
