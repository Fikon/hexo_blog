---
title: git笔记
date: 2017-01-05 10:55:25
tags: 那些年的坑
categories: [学习心得, 网络编程]
---

哇，真是心态爆炸，这运维小哥自己搭建的gitlab简直难用到死，UI用户体验差不说，最重要的是竟然会有
延迟，有延迟，有延迟！！兄弟，您 逗我呢？今天发现自己的push到dev分支的内容配置有点问题，就想着
回滚一下，`git reset *****`,然后一顿commit，pull，push和merge，然而似乎远端的dev对我有点意见，丝毫
不受我影响，R U kidding me?平常倒是还没注意，运维小哥还留着这么个彩蛋。然后小哥给我一顿教育，说
用sourceTree按照操作规范来就不会出问题||-_-||无奈，您那么帅，说啥都对。然后又是在sourceTree
的UI上面一顿的右键然后左键，好了，提交，推送，合并。认真你就输了，虽然我一顿操作，似乎dev仍旧是不领情，
反而是一堆的分支视图看着我想吐。于是再次跟小哥沟通，小哥无奈只能说由他来改。到了晚上我在去gitlab上面看，
呵呵，这原来的操作全部都生效了的，只不过不知道小哥这个玩意儿怎么搭的，搞得会有延迟，哇，出于对小哥的尊重，
吓得我赶紧再次复习下git用法。
<!--more -->

其实常用的git命令并不多（我用得少-_-||）,本文就按照一般的使用顺序来记录吧。
1. `git init`初始化一个新的空仓库（并不一定要在空文件夹中使用这个命令，有文件也可以），git会
  自动新建一个隐藏文件夹.git在这个文件夹中包含了各种版本控制信息，一般没啥事不用管他。
2. `git add` 这个命令用来告诉git你对哪些文件进行了操作，包括新建，更新会者是删除并将他们添加到暂存区(stage)。
    add后面可以跟文件，多个文件，也可以跟文件夹，`git add .`表示添加当前所有目录下所有文件，文件夹到stage。
3. `git commit -m 'hints'`这个命令跟在`git add`之后，将暂存区的文件提交到本地仓库，hints
可以是任意文字，但是最好是对本次修改的一个简要说明。
4.`gti status`查看当前仓库的状态，若是有更新在暂存区，比如新增，更新，删除则会有提示，若已经
commit过了之后则会告诉我们当前暂存区没有更新(类似：nothing added to commit but untracked files present)
5. `git diff file`查看文件与之前的区别，类似于linux的diff命令，不同的是diff针对的是同一个
的不同时期的差别，而linux diff则是两个不同文件的差别, `git diff head -- file`可以查看当前工作去中file文件与本地仓库最新版的差异，head代表本地仓库的最新版本
6. `git checkout -- file`在对file文件进行了更改之后，突然觉得这个更改是错误或者没必要的话，可以使用checkout撤销更改，回到上一次使用`git add`
   或者是`git commit`的状态，这里分两种情况，一个是更改了，但是还没add到stage, checkout的效果就是回到上一次commit的状态；另一个是已经add到stage了，并且之后又进行了修改
   这时候只能回到add时的状态了，不能撤销已经提交到stage的变更。要撤销的话需要用reset.
7. `git reset`回滚命令撤销stage的更改或者是将仓库回退到某个版本，当使用了add将变更提交到stage之后，checkout并不能撤销更改，这时候可以先用`git reset head file`
   然后再使用`git checkout -- file`这样就恢复到了上一次使用commit时的状态；reset还可以使仓库回退到某个版本`git reset --hard head^`表示回滚到上一个版本
`git reset --hard head^^`表示回滚到上两个版本，以此类推，回滚到上五十个版本的话就是`git reset--hard head^^...^^`当然这是不可能的，用`git reset --hard head^50`就好了，
如果忘记是前面第几个版本了，可以直接用提交时的ID来进行回退，那如何找到当次提交的ID呢，有办法，用`git log`就可以看到每次的历史提交信息以及当次提交的ID，于是`git reset --hard ID`就可以回滚到任意的一次提交了。好吧，干的漂亮，可是在回滚之后才发现其实没必要回滚呀，或者是回滚了到了太老的版本，这时候用log已经看不懂被删除的ID了怎么办？还好有`git reflog`这个后悔神器，可以看到所有的历史操作，
包括各种commit和reset操作的ID都能显示出来，reflog之后，再选一个恢复就好。
8. 增加远程仓库，在添加之前需要把SSH Key公钥提交到远端服务器上，首先生成密钥对`ssh-keygen -t rsa -C "user@host.com"`然后一路回车就好了，会在.ssh中生成两个密钥文件id_rsa和id_rsa.pub
   把公钥id_rsa.pub提交到服务器上可以通过SSH与服务器通信了。要增加远程仓库，在服务端应该有一个对应的git仓库，比如我们在远端建立了一个叫szyAndszy的仓库，可以用`git remote add origin user@host.com:path/szyAndszy.git`, path表示服务端仓库的路径
   这样origin就表示远端的szyAndszy仓库的代号了，下一步将本地仓库推送到远端，在推送之前一般先执行pull操作，把别人推送的更改跟我们的同步`git pull origin master`，然后`git push origin master`.master表示pull和push的都是master分支，若仅在master分支做更改的话，可以在第一次push的时候加上-u参数，则下次就不用加上master了。增加了远端仓库之后就可以跟小伙伴一起协同开发了。
9.克隆远端仓库，加入有个新的小伙伴要加入我们的开发团队了，可以直接在他电脑上执行对远端仓库的克隆操作即可获取最新版本的代码`git clone user@host.com:path/szyAndszy.git`
10. git分支，这是协同开发的神器，各个参与开发的小伙伴可以分别建立属于自己的分支，这样大家都在各自的分支上进行更改，提交互不影响，到最后在合并这些分支到主分支上面就ok了。`git branch bran`建立bran分支，`git checkout bran`可以切换到bran分支，刚刚建立并且要切换到分支的话，可以使用`git checkout -b bran`表示建立并且切换到bran分支，`git branch`可以查看当前仓库的所有分支，`git branch -d bran`删除bran分支，`git merge bran_name`表示将bran_name分支合并到当前分支，若无冲突的话就可以快速合并，有冲突则需要自己手动解决冲突才能合并。
