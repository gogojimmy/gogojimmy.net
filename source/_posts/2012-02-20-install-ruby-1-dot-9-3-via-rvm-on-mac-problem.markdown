---
layout: post
title: "[掃雷]在 Mac 上使用 RVM 安裝 Ruby 1.9.3 時錯誤"
date: 2012-02-20 01:54
comments: true
author: Jimmy Kuo
categories: [掃雷, RVM, Ruby 1.9.3]
---
## 環境：

* OS:Mac OSX 10.7
* RVM:1.8.2
* Ruby:1.9.3-p125
* Xcode:4.2

## 過程：

自從有了 RVM 後其實已經不太記得原本的 Ruby 怎麼裝了，某天想在公司的電腦安裝 1.9.3 時，一如往常的輸入 `rvm install 1.9.3` ，過程也一如往常的：
<pre>
Installing Ruby from source to: /Users/Jimmy/.rvm/rubies/ruby-1.9.3-p125, this may take a while depending on your cpu(s)...

ruby-1.9.3-p125 - #fetching
ruby-1.9.3-p125 - #extracted to /Users/Jimmy/.rvm/src/ruby-1.9.3-p125 (already extracted)
Fetching yaml-0.1.4.tar.gz to /Users/Jimmy/.rvm/archives
Extracting yaml-0.1.4.tar.gz to /Users/Jimmy/.rvm/src
Configuring yaml in /Users/Jimmy/.rvm/src/yaml-0.1.4.
Compiling yaml in /Users/Jimmy/.rvm/src/yaml-0.1.4.
Installing yaml to /Users/Jimmy/.rvm/usr
ruby-1.9.3-p125 - #configuring
ERROR: Error running ' ./configure --prefix=/Users/Jimmy/.rvm/rubies/ruby-1.9.3-p125 --enable-shared --disable-install-doc --with-libyaml-dir=/Users/Jimmy/.rvm/usr ', please read /Users/Jimmy/.rvm/log/ruby-1.9.3-p125/configure.log
ERROR: There has been an error while running configure. Halting the installation.
</pre>

幹這是哪招！ 拜求 Google 大神後找到的解法：

<pre>
$ rvm reinstall 1.9.3 --with-gcc=clang
</pre>

一樣無效！！

## 解法
這問題主要是出在 Xcode 4.2 以後 gcc 的位置就改了。

重新下載編譯 [GCC GCC Installer for OSX 10.7+, Version 2](https://github.com/kennethreitz/osx-gcc-installer/downloads)

然後就可以輕鬆的 `rvm install 1.9.3-p125` 了。
