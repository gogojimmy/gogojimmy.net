---
layout: post
title: "[掃雷]在Mac OSX 10.8.2 上安裝 Rmagick 失敗"
date: 2013-02-04 00:14
comments: true
categories: ruby gem rails rmagick
---

[Rmagick](https://github.com/rmagick/rmagick)是一個用Ruby寫成與[ImageMagick](http://www.imagemagick.org/script/index.php)溝通的界面，基本上只要有寫上傳圖片的功能就少不了他啊，今天在安裝一個新環境時照著平常以往的步驟安裝竟然一直失敗，奮力掃雷後來分享一下...

<!-- more -->


失敗的步驟：

```
brew install ImageMagick
gem install rmagick
uilding native extensions.  This could take a while...
ERROR:  Error installing rmagick:
	ERROR: Failed to build gem native extension.

        /Users/lifejimmy/.rvm/rubies/ruby-1.9.3-p374/bin/ruby extconf.rb
checking for Ruby version >= 1.8.5... yes
extconf.rb:128: Use RbConfig instead of obsolete and deprecated Config.
checking for /usr/local/bin/gcc-4.2... yes
checking for Magick-config... yes
checking for ImageMagick version >= 6.4.9... yes
checking for HDRI disabled version of ImageMagick... yes
checking for stdint.h... yes
checking for sys/types.h... yes
checking for wand/MagickWand.h... yes
checking for InitializeMagick() in -lMagickCore... no
checking for InitializeMagick() in -lMagick... no
checking for InitializeMagick() in -lMagick++... no
Can't install RMagick 2.13.1. Can't find the ImageMagick library or one of the dependent libraries. Check the mkmf.log file for more detailed information.

*** extconf.rb failed ***
Could not create Makefile due to some reason, probably lack of
necessary libraries and/or headers.  Check the mkmf.log file for more
details.  You may need configuration options.

Provided configuration options:
	--with-opt-dir
	--with-opt-include
	--without-opt-include=${opt-dir}/include
	--with-opt-lib
	--without-opt-lib=${opt-dir}/lib
	--with-make-prog
	--without-make-prog
	--srcdir=.
	--curdir
	--ruby=/Users/lifejimmy/.rvm/rubies/ruby-1.9.3-p374/bin/ruby
	--with-MagickCorelib
	--without-MagickCorelib
	--with-Magicklib
	--without-Magicklib
	--with-Magick++lib
	--without-Magick++lib


Gem files will remain installed in /Users/lifejimmy/.rvm/gems/ruby-1.9.3-p374/gems/rmagick-2.13.1 for inspection.
Results logged to /Users/lifejimmy/.rvm/gems/ruby-1.9.3-p374/gems/rmagick-2.13.1/ext/RMagick/gem_make.out
```

這問題的發生原因是由於[Rmagick](https://github.com/rmagick/rmagick)還不完整支援最新的[ImageMagick](http://www.imagemagick.org/script/index.php)，因此你必須手動fix:

```
$ cd /usr/local/Cellar/imagemagick/6.8.0-10/lib
$ ln -s libMagick++-Q16.7.dylib   libMagick++.dylib
$ ln -s libMagickCore-Q16.7.dylib libMagickCore.dylib
$ ln -s libMagickWand-Q16.7.dylib libMagickWand.dylib
```

考慮到[Rmagick](https://github.com/rmagick/rmagick)上次更新是兩年前的事情，或許現在開始可以考慮改用[MiniMagick](https://github.com/minimagick/minimagick)了。

參考：
[Error installing Rmagick on Mountain Lion](http://stackoverflow.com/questions/13942443/error-installing-rmagick-on-mountain-lion/13960185#13960185)

