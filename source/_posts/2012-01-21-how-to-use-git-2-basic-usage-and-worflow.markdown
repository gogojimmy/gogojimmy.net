---
layout: post
title: "Git 教學(2)：Git Branch 的操作與基本工作流程"
date: 2012-01-21 17:46
comments: true
Author: Jimmy Kuo
categories: Git
---

這是一篇系列教學文：

1. [Git 教學(1)：Git的基本使用](http://blog.gogojimmy.net/2012/01/17/how-to-use-git-1-git-basic/)
2. Git 教學(2)：Git Branch 的操作與基本工作流程

## Git branch
branch (分支)應該是 Git 最重要的技能了，在一個多人專案的開發過程中我們有時候要開發新功能，有時候是要修正某個Bug，有時候想要測試某個特異功能能不能 work ，這時候我們通常都會從主 branch 再開出一條新的 branch 來做，這支新開的 branch 會帶著你的主 branch 目前的最新狀態，當你完成你所要開發的新功能/ Bug 修正後確認沒問題就再把它 merge(合併)回主 Branch ，如此便完成了新功能的開發或是 Bug 的修正，因此每個人都可以從主 branch 拉一條新的 branch 來做自己想做的事，再來我們好好了解一下 branch 的使用。

<!--more-->

了解 branch 最好的方法就是有圖像可以看，你可以用任何你已安裝的GUI來查看，但在你安裝 Git 的時候其實同時也安裝了最基本的 Git GUI 叫做 gitk，你可以使用 `gitk --all` 這個指令來呼叫他，此時你應該能看到以下的畫面：

![gitk](https://lh3.googleusercontent.com/-0KUqcz4fH_I/TxMcZZU2VsI/AAAAAAAAA8Q/bNlXOjlN3NU/s640/2012-01-16_0233.jpg)

點擊左上方區塊的 commit 節點你可以看到當次 commit 的詳細資料，例如作者以及他的更新記錄，但你會發現這時候終端機是無法輸入的，因此請你先關掉 gitk ，改打指令 `gitk --all &` 來讓 gitk 在背景執行。

`git branch` 這個指令可以列出所有的 branch 並告訴你目前正在哪個 branch：
<pre>
$ git br
* master
  develop
  feature/test
</pre>
上面的訊息告訴我們在這個 Git repository裡有3支 branch ，而你目前正在 master branch 上。假設我們現在要開一支新的 branch 叫做 cat ，使用 `git branch` 來幫助你開一支新的 branch
<pre>
$ git branch cat
$ git branch
  cat
* master
</pre>
上面我們開了一支新的 branch 叫做 cat ，使用 `git branch` 再查看一次發現已經多了這支新的 branch了，這時候你去查看你的 gitk 的圖像狀態會發現像下圖一樣，新的 branch cat 與 master 在同一條水平線上，表示目前他們的狀態是一模一樣的。

![git branch](https://lh3.googleusercontent.com/-191UMSCyUtA/TxMjm_lkvbI/AAAAAAAAA9E/Fod45Hxj0gs/s800/2012-01-16_0305.jpg)

你應該也有發現，雖然我們建立了一個 cat 的 branch ，但其實我們所在的 branch 還是在 master branch，因此我們現在還需要切換過去，因此我們使用 `git checkout` 來切換：

<pre>
$ git checkout cat
Switched to branch 'cat'
</pre>

這樣就會從原本的 mater branch 切換到 cat branch 了。

接下來假設我正在 cat 這支 branch 做開發，因此新增一個檔案，加上一些內容，將它 add 到 stage 後再 commit 它。
<pre>
$ vim lib/cat.rb
$ git status
# On branch cat
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#
#	lib/cat.rb
nothing added to commit but untracked files present (use "git add" to track)
$ git add lib/cat.rb
$ git commit -m "Add Cat.rb"
[cat ea7d309] Add Cat.rb
 1 files changed, 3 insertions(+), 0 deletions(-)
 create mode 100644 lib/cat.rb
</pre>

![cat branch](https://lh4.googleusercontent.com/-OHOBrffzDXM/TxSumAE7wII/AAAAAAAAA9Q/pjQaLiDStVw/s640/2012-01-17_0709.jpg)

上面的流程你已經很熟悉了，接下來我們再切換到原本的 master branch ，這時候你會發現剛剛在 cat branch 新增的 cat.rb 檔案已經不見了。

<pre>
$ git checkout master
Switched to branch 'master'
Your branch is ahead of 'origin/master' by 1 commit.
</pre>

![master branch](https://lh6.googleusercontent.com/-_cHj5HPuUk4/TxSvXFQl-mI/AAAAAAAAA9Y/ZPKBHSfcBrM/s640/2012-01-17_0714.jpg)

Git 在我們切換 branch 的同時就會很聰明的會把我們的工作目錄更動成那個 branch 該有的狀態，如果你這時候切換到 GUI 去看，你會發現到與剛剛 cat 和 master branch 在同一條水平線上不同，cat branch現在已經比 master branch 再多出了一個 commit 的內容。

![gitk](https://lh6.googleusercontent.com/-WpmDiD9pCOw/TxSw6lRD7iI/AAAAAAAAA9k/YHaM6PuPJs4/s800/2012-01-17_0720.jpg)

現在我在切換到 cat branch 去增加更多的內容，一樣再將它 add 到 stage 後，再 commit 它。
<pre>
$ git checkout cat
Switched to branch 'cat'
$ mvim lib/cat.rb
$ git add lib/cat.rb
$ git commit -m "Add initializer"
[cat a3bce42] Add initializer
 1 files changed, 3 insertions(+), 1 deletions(-)
</pre>

![add initializer](https://lh4.googleusercontent.com/-ShoeQxd0BXU/TxSyncgOuII/AAAAAAAAA9w/2zRc_kzBswY/s640/2012-01-17_0728.jpg)

切到 GUI 來看的話你會發現現在 cat 這支 branch 比 master branch 要在多上兩個 commit 的內容。

![gitk](https://lh4.googleusercontent.com/-msnyGox6oqY/TxSzlC6TE2I/AAAAAAAAA94/agwGQVkMQQg/s800/2012-01-17_0732.jpg)

如果這時候我們在 master 上繼續開發會發生什麼事呢？ 我們現在切換到 master branch 並增加一個檔案及內容，照慣例 add 後 commit。
<pre>
$ git co master
Switched to branch 'master'
Your branch is ahead of 'origin/master' by 1 commit.
$ git add spec/animal_spec.rb
$ git commit -m "Another spec"
[master 7d72927] Another spec
 1 files changed, 2 insertions(+), 0 deletions(-)
</pre>

![master another spec](https://lh5.googleusercontent.com/-bI3T4UXtwzc/TxS96YN_C7I/AAAAAAAAA-E/BoVEpL0RZ9I/s640/2012-01-17_0816.jpg)

我在 master branch 的 animal_spec.rb 增加了一些內容，把它 commit 之後我們切換到 GUI 來看。

![master spec](https://lh4.googleusercontent.com/-qYJ-rOokSco/TxS_8bxidzI/AAAAAAAAA-Q/u8b2EXKfFng/s800/2012-01-17_0825.jpg)

我們發現現在 master branch 與 cat branch 已經產生分歧了，因為兩支 branch 都有了各自往後開發的 commit ，而且由於 master branch 最後一次的 commit 時間較新因此排列在最前面。

###Git rebase 整理現在的 branch
假設我們現在在 cat branch 的開發動作已經完畢，通常我們現在要做的事情會是將 cat branch 合併回 master branch，在開發流程上， master branch 就像是一個主要的 branch ，每個開發人員都是從 master branch checkout 出去一支新的 branch 做開發，在開發完畢後就再將開發完的 branch 合併回 master branch，因此 master branch 都會保有最新的開發好的狀態，一般在 Git 教學中會教你現在使用 `git merge` 這個指令來將兩個 branch 合併，但這邊我要先教你的是 `git rebase` 這個指令。

與 `git merge` 不同的是， `git rebase` 不單單只是將兩個不同的 branch 合併起來，而是將**某一支 branch 基於另一支 branch 的內容合併起來**，這是什麼意思？ 以我們的例子來說，我們在 cat branch 開發完了以後，這時候我們的 master branch 也有了其他開發者所合併回去的內容，換句話說現在的 master branch 與我們當初 checkout 出去的時候的狀態已經不同了，但我們會希望我們現在這支 cat branch 的內容就像是剛剛從 master branch checkout出來一樣乾淨，也就是說讓 cat branch 中保有 master branch 最新的狀態， `git rebase` 會基於 master branch 目前最後一次的 commit 內容再往後把你在 cat branch 上commit 的內容加上去，我們現在在 cat branch 輸入 `git rebase master` 來將 cat branch 基於 master branch 做 rebase。

<pre>
$ git rebase master
First, rewinding head to replay your work on top of it...
Applying: Add Cat.rb
Applying: Add initializer
</pre>

過程中沒有發生衝突，這時候我們到 GUI 看看現在的結果。

![git rebase](https://lh4.googleusercontent.com/-esENJUTDCjU/TxTGtd1yeLI/AAAAAAAAA-c/VXTyBX9B7EI/s800/2012-01-17_0848.jpg)

原先 cat branch 上的兩個 commit (Add Cat.rb 和 Add initializer) 已經合併到 master branch 最新的 commit (Another spec)，換句話說目前 cat branch 的內容就像是剛從 master branch 所 checkout 出來然後再加上自己的 commit，因此不同於 `git merge` 的線圖會把 cat branch 合併到 master branch ， 而是把原本的 cat branch 接到 master branch 因此只有一條線，當一個專案有很多的 branch 再做開發的時候會避免很多 branch 的線接來接去難以辨認。

**開發過程中，若你在開發的 branch 功能比較多， commit 的量也比較多時，建議使用rebase將你現在的 branch 整理過再合併回主幹，這樣會產生較漂亮的線圖**

若你想要看看目前的 branch 與其他 branch 有哪些差異，你可以使用 `git diff` 的指令去觀察，例如我現在想要看 master 跟 cat 這兩個 branch 的差異，我只要下：

<pre>
$ git diff cat master
diff --git a/lib/cat.rb b/lib/cat.rb
deleted file mode 100644
index 1227d26..0000000
--- a/lib/cat.rb
+++ /dev/null
@@ -1,5 +0,0 @@
-class Cat < Animal
-  def initialize
-    super
-  end
-end
</pre>

你就可以看到現在的 cat branch 跟 master branch 的差異在哪了。

如果我們開發完畢時，我們會把開發好的東西合併回 master 很自然的我們通常都會使用 `git merge` 這個指令來合併兩個branch

<pre>
$ git merge cat
Merge made by recursive.
 lib/cat.rb |    3 +++
 1 files changed, 3 insertions(+), 0 deletions(-)
 create mode 100644 lib/cat.rb
</pre>

這時候我們看圖的話會是這樣：

![git merge](https://lh4.googleusercontent.com/-JriNDcS61UI/TxmrJsBo7wI/AAAAAAAAA-s/o2_A-ju4zrE/s800/2012-01-21_0156.jpg)

可以看到我剛剛在 master branch 下了 `git merge cat` 這個指令來告訴 git 要 merge cat 到現在所在的 branch ，因此在圖上就看到了 cat branch 拉一條線回來合併到了 master 這個 branch 了，解釋這張圖的意思就是， cat branch 從 master branch 的 Another spec 這一次的 commit 分支出來後，自己產生了三次的 commit (Add Cat.rb、Add initializer、Rever "Add initializer") 然後合併到 master。

###Confict: 處理 Git 合併時的衝突：
很常發生的情況是再 merge 或是 rebase 的圖中產生了 convict (衝突)，這時候 Git 會停下來請你去處理，例如我們在 cat 和 master 的 branch 都對 lib/cat.rb 這支檔案做編輯，然後我們將他們 merge：

<pre>
$ git merge cat
Auto-merging lib/cat.rb
CONFLICT (content): Merge conflict in lib/cat.rb
Automatic merge failed; fix conflicts and then commit the result.
</pre>

你會看到 Git 告訴你在你合併的過程中在 lib/cat.rb 這支檔案發生了衝突，Git 不知道該怎麼處理因此要你去處理它，這時候我們打開這支檔案會看到這樣的情況：

![git confict](https://lh6.googleusercontent.com/-ZrCt7H7xBJ8/TxpugjL11TI/AAAAAAAAA_E/eN35v4t7qXE/s800/2012-01-21_1550.jpg)

<<<<<<<<<< HEAD 到 ========== 的中間區域是目前你所在 branch 的 commit 內容，而從 =========== 到 >>>>>>>>>>> cat 則是你要合併的 cat branch 的內容，你必須做出決定看是要兩個都留下，或是選一個，或是改成你想要的內容，改好後記得要將 Git 自動產生的 <<< 、 =====、 <<< 的內容都刪除，修改完畢後存檔，將剛剛修改過的檔案再使用 `git add` 加入 stage ，將所有的衝突都修正完畢後就使用 `git commit` 提交一次 commit，由於這次的 commit 是在處理 merge 時的衝突，因此 Git 很聰明的已經幫我們加上了一些預設的訊息 "Merge branch 'cat'"， commit 提交後就會看到合併成功的訊息了。

<pre>
$ git add lib/cat.rb
$ git commit
[master c37c9e3] Merge branch 'cat'
</pre>

**發生 confict 時的處理步驟**

1. **將發生 confict 的檔案打開，處理內容( 別忘了刪除<<<、===、>>> )。**
2. **使用 `git add` 將處理好的檔案加入 stage。**
3. **反覆步驟 1~2 直到所有 confict 處理完畢。**
4. **`git commit` 提交合併訊息。**
5. **完成**

**講到這裡我們再來整理一下工作流程，讓大家再複習一下 git 的使用:**

1. **在專案中會有一條主 branch 是大家將開發好或是修好的東西合併回去的對象，所有要開發的新功能或是修 bug 都是從主 branch 拉出一條新的 branch 去工作。**
2. **當你完成一個階段性的任務時，將你剛剛 所新增的內容使用 `git add` 加入到 stage 的狀態，並且使用 `git commit` 加上 commit 的訊息來提交一次的 commit。**
3. **反覆動作 2 直到你完成這支 branch 的主要目的(新功能/修 bug )，若這時你離主 branch 已經有一段時間，或是確定主 branch 上已經有新的 commit ，使用 `git rebase` 將自己的分支整理然後使用 `git merge` 合併回主 branch，反之則是直接使用 `git merge` 將自己 branch 的內容合併回去。**

## Git reset 取消上一次的操作

### 取消 merge
版本控制最大的好處之一就是讓你永遠可以後悔，因此我們常會希望把已暫存的檔案、已提交的 commit 或是已合併的 branch 取消修改，這時候我們可以使用 `git reset` 這個指令來幫助我們，像現在我若是想要取消剛剛的 merge 動作，我只要下：

<pre>
$ git reset --hard ORIG_HEAD
HEAD is now at c126ff9 Config initialze
</pre>

這時候再回去看圖你會發現已經回到合併前的樣子了：

![git reset](https://lh5.googleusercontent.com/-2tyXpiUJKmU/Txm0r4vgeDI/AAAAAAAAA-4/k2s_JJlyj1E/s800/2012-01-21_0238.jpg)

### 取消已暫存的檔案
有時候手殘不小心將還沒修改完的檔案使用 `git add` 加入了 stage ，這時候可以使用 `git reset HEAD <file>` 來將這支檔案取消 stage：

![git reset](https://lh5.googleusercontent.com/-LI6SAuanFeQ/Txp07PZatSI/AAAAAAAAA_Q/eXB_6EVU1Ck/s800/2012-01-21_1618.jpg)

你可以看到我使用 `git add` 將檔案加入 stage 後，在我的 status 狀態顯示 lib/cat.rb 這支檔案現在已經準備好被 commit ，但這時我使用了 `git reset HEAD` 將這支檔案取消 stage，再使用 status 查看時它就變回一支還沒加入 stage 的檔案了。

### 取消修改過的檔案
連續剛剛的情況，若是我想完全放棄這次修改 (將檔案狀態回復到最新的一次 commit 時的狀態)，我可以使用 `git checkout -- <file>` 來回復這支檔案：

<pre>$ git checkout -- lib/cat.rb</pre>

取消變更不會有任何訊息，但這時你去看檔案會發現他已經回復成沒修改過時的模樣了。

### 修改上一次的commit
手誤打太快， commit 訊息打錯時，我們可以使用 `git commit --amend` 來幫助我們重新修改：

![git reset](https://lh5.googleusercontent.com/-EWew34dGcNs/Txp4qkIcZwI/AAAAAAAAA_c/cAgGz7WWS5k/s800/2012-01-21_1634.jpg)
![git reset](https://lh5.googleusercontent.com/-f2T7R1Bq7oM/TxqEN_EfY9I/AAAAAAAAA_o/IHlCkmOX1vk/s800/2012-01-21_1723.jpg)

在上面我想要修改打錯字的 commit 訊息 "Cat initiae"，因此我使用 `git commit --amend` 來修改成正確的訊息。

### 強制回復到上一次 commit 的版本
有時候我們想要放棄所有修改回到 commit 時的狀態，這時候我們可以下 `git reset --hard HEAD` 來回復，HEAD 參數可以加上一些變化，例如 HEAD^ 表示目前版本的上一個版本 HEAD~2 則是再上一個，因此你可以自由的跳回去之前的狀態。

### git reset, hard 與 soft 的差異
你可能會在這邊感到疑惑，在使用 `git reset` 的時候都會看到一個 soft 或是 hard 的參數，這代表什麼樣的意義？基本上在使用 `git reset` 的時候，都會把目前狀態回復到你想回復的版本，但若是不加參數的情況，會把你做過的修改仍然保留，但是，若是加上 --soft 參數，則會把做過的修改加入 stage ，若是加上 hard 參數的話則是把做過的修改完全刪除，回到那個版本原本的樣子。
