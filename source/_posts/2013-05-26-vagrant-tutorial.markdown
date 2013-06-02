---
layout: post
title: "[教學]使用Vagrant練習環境佈署"
date: 2013-05-26 15:09
comments: true
categories:
---

最近對Rails的佈署有更深一層的體悟，打算花點時間將佈署心得整理成文章，預計大概會在2050年前完成這部大作，這邊決定先發布序章，就是教你怎麼使用Vagrant來打造自己的測試機器。

<!-- more -->


## 為什麼要用Vagrant

答案很簡單，因為開遠端機器練習佈署或是機器架構又慢又麻煩又要錢，而且玩壞了或是環境髒了又得重灌又很慢，如果你想我一樣最近在玩Chef-Server，實驗多機器環境架構，例如一台Web Load Balancer、5台Application Server、1台Master Database + 1台Slave Database、1台Data Analytics Server、1台 Service Monitor Server，1台Redis Server，聽起來有沒有很牛，但這一切都只要在你那台用來開發的電腦就做的到，告訴我為何你還不用Vagrant呢？

## Vagrant Basic

### 下載安裝Vagrant與Virtual Box
Vagrant背後用的是Virtual Box作為虛擬機器，Vagrant只是一個讓你可以方面做設定來開你想要的虛擬機器的方便工具，所以你必須先安裝Vagrant和Virtual Box，Virtual Box你可以在[Virtual Box官網](https://www.virtualbox.org/)下載適合你平台的版本，而Vagrant你可以在[Vagrant官網](http://www.vagrantup.com/)下載打包好的版本，或是如果你跟我一樣是個玉樹臨風的Rubist，你可以打開我們最愛的小黑視窗輸入

```
$ gem install vagrant
```

> Vagrant 1.1後已經不支援使用Gem來安裝了，據說是因為Dependecy太多他受不了了，詳細可以參考他們的[說明](http://mitchellh.com/abandoning-rubygems)，感謝ihower的提示

## 開始使用Vagrant 新增作業系統
當你已經安裝好Virtual Box以及Vagrant後，你要開始思考你想要在你的VM上使用什麼作業系統，一個打包好的作業系統環境在Vagrant稱之為Box，也就是說每個Box都是一個打包好的作業系統環境，當然網路上什麼都有，你不用自己去找作業系統，[vagrantbox.es](vagrantbox.es)上面就有許多大家熟知且已經打包好的作業系統，你只需要下載就可以了，為你的Vagrant增加一個Box很簡單

```
$ vagrant box add {你想要的Box名稱} {下載網址}
```

例如我想要下載我最愛的<del>也只會這個的</del>Ubuntu，我只要在[vagrantbox.es](vagrantbox.es)挑好我想要的版本，一樣在我們的小黑視窗輸入

```
$ vagrant box add {Ubuntu-12-10} {http://cloud-images.ubuntu.com/quantal/current/quantal-server-cloudimg-vagrant-amd64-disk1.box}
```

Vagrant就會開始下載這個Box，你可以用`vagrant box list`這個指令看到你所擁有的所有Box，想像就是你的書架上多了一片Ubuntu 12.10的安裝光碟，以後要安裝機器就是用這的安裝就可以了，有了Box以後，我們要產生一個設定檔來設定我們的虛擬機器，這個檔案可以透過指令`vagrant init Box名稱`來產生，你可以在你的專案中或是另外開個練習用的資料夾輸入，這時候你的資料夾終究會有一個名稱為`vagrantfile`的檔案，這個檔案就是所有魔法的開始

```
~/Dropbox/Projects/Personal/vagrant » vagrant init ubuntu-12-10                                                                                             gogojimmy@MBP
A `vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant.
```

### 讓VM動起來
我們晚一點再提設定檔的部份，讓我們先把VM跑起來，要讓VM跑起來的指令是`vagrant up`

```
$ vagrant up                                                                                                      gogojimmy@MBP
[default] VM already created. Booting if it's not already running...
[default] Clearing any previously set forwarded ports...
[default] Forwarding ports...
[default] -- 22 => 2222 (adapter 1)
[default] Creating shared folders metadata...
[default] Clearing any previously set network interfaces...
[default] Booting VM...
[default] Waiting for VM to boot. This can take a few minutes.
[default] VM booted and ready for use!
[default] The guest additions on this VM do not match the install version of
VirtualBox! This may cause things such as forwarded ports, shared
folders, and more to not work properly. If any of those things fail on
this machine, please update the guest additions and repackage the
box.

Guest Additions Version: 4.1.18
VirtualBox Version: 4.2.12
[default] Mounting shared folders...
[default] -- v-root: /vagrant
```

看到以上這段指令跑完後，讓我們用`ssh`連線到機器

```
$ vagrant ssh
```

現在你已經在虛擬機器上了，登入的用戶是預設的`vagrant`，現在你可以開始玩壞你的機器了，在VM中有個`/vagrant`的資料夾會與你`host`機器的vagrant設定檔所在的資料夾共享資料，現在你可以照你的習慣把機器的環境安裝起來，例如以我來說我會先安裝Ruby以及一些基本的系統套件，你可以在我的[Gist](https://gist.github.com/gogojimmy/5523985)上看到這個裝機步驟，等我一下，讓我先把我的環境先安裝好，你可以先去玩一下Candy Crush或是看一下Facebook

```
vagrant@vagrant-ubuntu-quantal-64:~$ curl -L https://gist.github.com/gogojimmy/5523985/raw/b9d777bc380ee791c2f4534e9261b4b99289ed9f/bootstrap-chef-solo.sh | sh
```

### 將習慣的環境打包成Box
我的環境大概要裝15分鐘左右，有人說一個程式設計師的人生花最多時間的一件事情就是等，這個說法真是一點也不為過，每次裝機都從環境重新開始Build也不是辦法，我們要讓人生有更多的時間去處理更多的事情，所以我們可以把一個已經Build好的環境打包成一個我們自己的Box，以後我們只要直接使用這個打包好的版本就可以了，因此讓我們幫自己的人生省點時間，速速登出VM來打包這個Box

```
vagrant@vagrant-ubuntu-quantal-64:~$ exit
logout
Connection to 127.0.0.1 closed.
------------------------------------------------------------
$ vagrant package                                                                                                 gogojimmy@JoycetekiMacBook-Pro
[default] Attempting graceful shutdown of VM...
[default] Clearing any previously set forwarded ports...
[default] Creating temporary directory for export...
[default] Exporting VM...
[default] Compressing package to: /Users/gogojimmy/Dropbox/Projects/Personal/vagrant/package.box
```

`vagrant package`這個指令會在你目前的資料夾下建立一個`vagrant.box`的Box檔案，這時候我們跟剛剛一樣把它加入到我們的Box List中，以後我們就可以快速使用這個Box就好了！除此之外，可以自定Box的意義還有讓你的團隊都能用VM來擁有自己的Staging環境，例如在Rails專案中我們也可以建立一個Vagrant的設定檔來做一個給開發人員測試用的Staging環境，這時候你就可以指定好你自定的機器設定，確保每個開發人員都能擁有一樣的環境來進行開發。

```
$ vagrant box add gogojimmy-ubuntu-12-10 package.box
$ vagrant list
```

###Vagrant基本設定

####設定VM的名稱及記憶體
用你最喜歡的編輯器打開`vagrantfile`，`vagrantfile`是個有著詳細解釋的設定檔，非常建議妳好好看過一遍來了解每個設定，當然我也知道今天你會來看這篇文章就表示你根本懶得看英文，就讓我來幫你一步步走過，在這個檔案中所有的設定都被`Vagrant::Config.run`的Block包起來，在一開始只會有box的設定：

``` ruby
config.vm.box = "gogojimmy-ubuntu-12-10"
```

這告訴了Vagrant要去套用哪個Box作為環境，也就是你一開始輸入`varant init Box名稱`時所指定的Box，如果沒有輸入Box名稱的話就會是預設的`base`，Virtual Box本身提供了`VBoxManage`這個command line tool讓你可以設定你的VM，用`modifyvm`這個指令讓你可以設定VM的名稱及記憶體大小等等，這裡說的名稱指的是在Virtual Box中顯示的名稱，我們也可以在`vagrantfile`中進行設定，在你的`vagrantfile`中加入這行

``` ruby
config.vm.customize ["modifyvm", :id, "--name", "gogojimmy", "--memory", "512"]
```

這行設定檔意思就是呼叫VBoxManage的`modifyvm`的指令，設定VM的名稱為`gogojimmy`，而設定VM的記憶體大小為512MB，你可以照這這種作法為你的VM設定好不同的設定。

####設定Hostname以及Port forward
設定`hostname`非常簡單，設定中加入下面這行就好

``` ruby
config.vm.host_name = "gogojimmy-app"
```

設定`hostname`非常重要，有很多服務都仰賴著`hostname`來做為辨識，例如`Puppet`或是`Chef`，一般一些監控服務像是New Relic之類的也都是以`hostname`來做為辨識，設定那麼簡單，就別偷懶吧，再來看到下面這行

``` ruby
config.vm.forward_port 80, 8080
```

這個設定非常的厲害，這行的意思就是把Host機器上8080 port傳來的東西`forward`到VM跑的 80 port的服務，例如當你練習Deploy到VM上時，在瀏覽器打開http://localhost:8080時就會傳入VM裡跑80 port的服務像是Nginx或是Apache，因此我們可以透過這個設定來幫助我們去設定Host與VM間，或是VM與VM間的溝通，像是MySQL通常跑3306之類的

####設定網路橋接方式
Vagrant有兩種橋接方式是，一種是host only，意思是說在你電腦同個區網中的其他電腦是看不到你的VM的，只有你一個人自High，另一種是Bridge，當然就是說VM會跟你區網的router去要一組IP，區網中的其他電腦也都能看到他，一般來說因為開VM的情況都是自High居多，因此我們在設定上都是設定host only：

``` ruby
config.vm.network :hostonly, "192.168.33.10"
```

這邊將網路設定成`hostonly'，並且指定一組IP位址，IP位址的設定會建議不要使用`192.168.*.*`的設定，因為很有可能會跟你區網的IP衝突，你可以改使用像是`33.33.*.*`的設定。

> 更改vagrantfile的設定後，記得要用`vagrant reload`的指令重開VM讓VM可以用新的設定檔跑起來

###讓我們開始打造多機器環境
重頭戲來了，前面的一切都是為了今天鋪陳，現在我們要建立多個VM跑起來，並且讓他們互相溝通，有人跑Application、有人跑DB、有人跑Memcached，這一切在Vagrant中非常簡單，跟剛剛的設定都一樣，你只需要指定好機器的角色就可以了，讓我們再次打開我們的設定檔來設定一台APP Server加上一台DB Server：

``` ruby
config.vm.define :app do |app_config|
    app_config.vm.customize ["modifyvm", :id, "--name", "app", "--memory", "512"]
    app_config.vm.box = "ubuntu-12-10"
    app_config.vm.host_name = "app"
    app_config.vm.network :hostonly, "33.33.13.10"
end
config.vm.define :db do |db_config|
  db_config.vm.customize ["modifyvm", :id, "--name", "db", "--memory", "512"]
  db_config.vm.box = "ubuntu-12-10"
  db_config.vm.host_name = "db"
  db_config.vm.network :hostonly, "33.33.13.11"
end
```

這邊的設定就像是剛剛在設定的部份教的一樣，只是我們使用了`:app`以及`:db`分別做了兩個VM的設定，並且給予不同的`hostname`和IP，設定好了以後再使用`vagrant up`將機器跑起來：

```
$ vagrant up
[app] Importing base box 'ubuntu-12-10'...
[app] Matching MAC address for NAT networking...
[app] Clearing any previously set forwarded ports...
[app] Forwarding ports...
[app] -- 22 => 2222 (adapter 1)
[app] -- 80 => 8080 (adapter 1)
[app] Creating shared folders metadata...
[app] Clearing any previously set network interfaces...
[app] Preparing network interfaces based on configuration...
[app] Running any VM customizations...
[app] Booting VM...
[app] Waiting for VM to boot. This can take a few minutes.
[app] VM booted and ready for use!
[app] Configuring and enabling network interfaces...
[app] Setting host name...
[app] Mounting shared folders...
[app] -- v-root: /vagrant
[db] Importing base box 'ubuntu-12-10'...
[db] Matching MAC address for NAT networking...
[db] Clearing any previously set forwarded ports...
[db] Fixed port collision for 22 => 2222. Now on port 2200.
[db] Fixed port collision for 22 => 2222. Now on port 2201.
[db] Forwarding ports...
[db] -- 22 => 2201 (adapter 1)
[db] Creating shared folders metadata...
[db] Clearing any previously set network interfaces...
[db] Preparing network interfaces based on configuration...
[db] Running any VM customizations...
[db] Booting VM...
[db] Waiting for VM to boot. This can take a few minutes.
[db] VM booted and ready for use!
[db] Configuring and enabling network interfaces...
[db] Setting host name...
[db] Mounting shared folders...
[db] -- v-root: /vagrant
```

看到上面的訊息跑完後，你就可以跟剛剛一樣使用`ssh`連到VM裡，但這次不同的是你要加上你所指定的角色告訴你要連線的機器是哪一台：

```
$ vagrant ssh app
vagrant@app:~$

$ vagrant ssh db
vagrant@db:~$
```

是不是很酷！！再來我們來驗證一下VM之間的連線，讓我們使用`ssh`登入`db`的機器，然後在`db`的機器上使用`ssh`來連線到`app`的機器(預設密碼就是vagrant)：

```
$ vagrant ssh db

vagrant@db:~$ ssh 33.33.13.10
The authenticity of host '33.33.13.10 (33.33.13.10)' can't be established.
ECDSA key fingerprint is a7:71:36:4c:01:4a:38:a2:fc:fa:ea:d7:67:63:3c:40.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '33.33.13.10' (ECDSA) to the list of known hosts.
vagrant@33.33.13.10's password:

vagrant@app:~$
```

看到了嗎，VM之間的溝通也是沒有問題的！，你現在可以開始好好思考你偉大的`Infrastructure`，讓你的程式跑在多機器的環境中，如果你對於Infrastructure不熟悉，Amazon有提供了不少[範例](http://aws.amazon.com/architecture/)可以參考，想像力就是你的超能力，現在唯一侷限你的只會是你的電腦記憶體了，不要開到跑不動都不會有事的，今天就開始用Vagrant練習你的機器佈署吧！

> 記得設定多台機器的時候，剛剛原先在單台機器的設定必須先清除或是註解掉，如果剛剛的VM還在跑，記得要用`vagrant halt`先關機
