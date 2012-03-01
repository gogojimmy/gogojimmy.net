---
layout: post
title: "Git 情境劇"
date: 2012-02-29 00:14
comments: true
Author: Jimmy Kuo
categories: Git
---
## Git 情境劇
這篇主要是給自己做個記錄，因為 Git 指令實在太多了...

* 如何安裝 Git
    * Mac : 安裝 [Homebrew](http://mxcl.github.com/homebrew/)

            brew install git
    * Linux(Debian) : `apt-get install git-core`
    * Linux(Fedora) : `yum install git-core`
    * Windows : 下載安裝 [msysGit](http://code.google.com/p/msysgit)
* 如何設定 Git
    * Mac : [Set Up Git on Mac](http://help.github.com/mac-set-up-git/)
    * Linux : [Set Up Git on Linux](http://help.github.com/linux-set-up-git/)
    * Windows : [Set up Git on Windows](http://help.github.com/win-set-up-git/)

* 如何開始一個 Git Respository
    * 在專案底下使用 `git init` 開始一個新的 Git repo.
    * 使用 `git clone` 複製一個專案

* 如何將檔案加入 Stage
    * 使用 `git add` 將想要的檔案加入 Stage.
    * `git add .` 會將所有編修過的檔案加入 Stage (新增但還沒 Commit 過的檔案並不會加入)

* 如何將檔案從 Stage 中移除(取消add)
    * `git reset HEAD 檔案名稱`

* 如何將檔案提交(commit)
    * 使用 `git commit`會將 Stage 狀態的檔案做 Commit 動作
    * `git commit -m "commit訊息"` 可以略過編輯器直接輸入 commit 訊息完成提交。
    * `git commit -am "commit訊息"` 等同於先`git add .`後略過編輯器提交 commit。

* 如何修改/取消上一次的 commit
    * `git commit --amend` 修改上一次的 commit 訊息。
    * `git commit --amend 檔案1 檔案2...` 將檔案1、檔案2加入上一次的 commit。
    * `git reset HEAD^ --soft` 取消剛剛的 commit，但保留修改過的檔案。
    * `git reset HEAD^ --hard` 取消剛剛的 commit，回到再上一次 commit的 乾淨狀態。

* 分支基本操作(branch)
    * `git branch` 列出所有本地端的 branch。
    * `git branch -r` 列出所有遠端的 branch。
    * `git branch -a` 列出所有本地及遠端的 branch。
    * `git branch "branch名稱"` 建立一個新的 branch。
    * `git checkout -b "branch名稱"` 建立一個新的 branch 並切換到該 branch。
    * `git branch branch名稱 起始點` 以起始點作為基準建立一個新的 branch，起始點可以是一個 tag，branch 或是 commit。
    * `git branch --track branch名稱 遠端branch` 建立一個 tracking 遠端 branch 的 branch，這樣以後 push/pull都會直接對應到該遠端的branch。
    * `git branch --set-upstream branch 遠端branch` 將一個已存在的 branch 設定成 tracking 遠端的branch。
    * `git branch -d "branch 名稱"` 刪除 branch。
    * `git -r -d 遠端branch` 刪除一個 tracking 的遠端 branch，例如`git branch -r -d wycats/master`
    * `git push repository名稱 :遠端branch` 刪除一個 repository 的 branch，通常用在刪除遠端的 branch，例如`git push origin :old_branch_to_be_deleted`。
    * `git checkout branch名稱` 切換到另一個 branch(所有修改過程會被保留)。

* 遠端操作(remote)
    * `git remote add remote名稱 remote網址` 加入一個 remote repository，例如 `git remote add github git://github.com/gogojimmy/test.git`
    * `git push remote名稱 :branch名稱` 刪除遠端 branch，例如 `git push origin :somebranch`。
    * `git pull remote名稱 branch名稱` 下載一個遠端的 branch 並合併(注意是下載遠端的 branch 合併到目前本地端所在的 branch)。
    * `git push` 類似於 pull 操作，將本地端的 branch 上傳到遠端。

* 合併操作(merge)
    * `git merge branch名稱` 合併指定的 branch 到目前的 branch。
    * `git merge branch名稱 --no-commit` 合併指定的 branch 到目前的 branch 但是不會產生合併的 commit。
    * `git cherry-pick SHA` 將某一個 commit 的內容合併到目前 branch，指定 commit 是使用該 commit 的 SHA 值，例如 `git cherry-pick 7300a6130d9447e18a931e898b64eefedea19544`。

* 暫存操作(stash)
    * `git stash` 將目前所做的修改都暫存起來。
    * `git stash apply` 取出最新一次的暫存。
    * `git stash pop` 取出最新一次的暫存並將他從暫存清單中移除。
    * `git stash list` 顯示出所有的暫存清單。
    * `git stash clear` 清除所有暫存。

* 常見問題：
    * 我的 code 改爛了我想全部重來，我要如何快速回到乾淨的目錄?
        * `git reset --hard` 這指令會清除所有與最近一次 commit 不同的修改。
    * merge 過程中發生 confict 我想放棄 merge，要如何取消 merge？
        * 一樣使用 `git reset --hard` 可以取消這次的 merge。
    * 如何取消這次的 merge 回到 merge 前的狀態?
        * `git reset --hard ORIG_HEAD` 這指令會取消最近一次成功的 merge 以及所有你在這次 merge 後所做的修改。
    * 如何回復單獨檔案到原本 commit 的狀態?
        * `git checkout 檔案名稱` 這指令會將已經被修改過的檔案回復到最近一次 commit 的樣子。
