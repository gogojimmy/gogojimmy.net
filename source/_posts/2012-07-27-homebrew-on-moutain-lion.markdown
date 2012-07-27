---
layout: post
title: "安裝 Moutain Lion 後，一些掃雷步驟(Homebrew)"
date: 2012-07-27 22:11
comments: true
categories:
---

身為一個勇者，升級作業系統永遠都是搶第一的，當然免不了遇到地雷炸一下，大概掃了一下雷後，分享一下我的山獅更新步驟。

1. 從 App Store 下載安裝山獅。
2. 從 App Store 更新你的 Xcode，更新完畢後在設定 > 下載 > 安裝 Command Line Tools
3. 因為[山獅已經移除了 X11](http://support.apple.com/kb/HT5293?viewlocale=en_US&locale=en_US)，請下載安裝 [XQuartz](http://xquartz.macosforge.org/landing/)
4. 有些人安裝了 Xcode 的 Command Line Tools 後一樣找不到 GCC (知道原因的拜託告訴我)，你可以下載安裝人家打包好的 [GCC for OSX](https://github.com/kennethreitz/osx-gcc-installer/)
5. 更新升級 Homebrew 中的套件 `brew update && brew upgrade`

這樣應該就能正常使用了

其他：
* 你可能會看到會出現這樣的訊息 `WARNING: Nokogiri was built against LibXML version 2.8.0, but has dynamically loaded 2.7.8`

在 StackOverFlow 找到的[解法](http://stackoverflow.com/questions/11452380/warning-nokogiri-was-built-against-libxml-version-2-7-3-but-has-dynamically-lo)：
```
gem uninstall nokogiri libxml-ruby

brew uninstall libxml2
brew install libxml2 --with-xml2-config
brew link libxml2

brew install libxslt
brew unlink libxslt

gem install nokogiri -- --with-xml2-include=/usr/local/Cellar/libxml2/2.8.0/include/libxml2/ --with-xml2-lib=/usr/local/Cellar/libxml2/2.8.0/lib/ --with-xslt-dir=/usr/local/Cellar/libxslt/1.1.26/
```
