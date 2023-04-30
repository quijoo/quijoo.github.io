# [学习记录] git 分布式版本管理系统


常用命令速查 [git-cheat-sheet](/git-cheat-sheet.pdf)

## 基础 
git status 查看信息 
git diff   查看差异 
git log    查看commit 历史，由近到远  提交的版本号是一个哈希值 
使用 --pretty-oneline 将一个commit打印到一行


## 版本库 & 工作区  

git reset --hard HEAD^ 是回退到上一个版本，

在reset回退版本时，版本链会断掉，使用 git log 看不到之前的提交记录了， 可以找窗口之前的记录，得到某个commit的Id 并且使用 git reset --hard 1094a 回退， 这里的1094a 是id的前几位！！  

如果窗口被关闭了还可以使用 git reflog查看每一次命令， 所以在 git中有一个很重要的是一定要为 commit 添加注释！！

 HEAD 指针指向当前版本！   
 
Git会追踪被add过的文件的所有改变，没有被add的文件只会被标记为untrack  

git diff HEAD -- readme.txt可以查看变动！这里的变动是 commit 与 工作区 的差异！  

git checkout -- file 重新检出，丢弃工作区的修改 checkout 要么回到暂存区，要么回到版本库，总之回到最近的修改 这里的--指定文件！如果不加就是切换分支的命令！！  

git reset HEAD <file> 可以撤销掉暂存区的修改

删除文件 git rm git commit 
如果是误删，就git checkout -- file 来撤销工作区的修改 checkout 的本质是用版本库替换工作区！


## 远程仓库 
git remote add origin 地址  的含义是将当前仓库和远程仓库建立连接，其中 origin是远程仓库名字？？？  
第一次推送代码时，使用 git push -u origin master， origin是指定了远程仓库名字 这里的 master 是本地的master分支！！-u 初次使用时添加，意思是将他们关联起来！  
git ls-files [--cached --delete --modified --other --stage] 
git push [-f 强制推送，可能覆盖] git push -u origin [branch] 创建新分支时使用，这会创建一个与本地分支 branch 对应的上游分支 
git push -all 这会 push 所有分支 
git push --tag 发布尚未在远程仓库中的标签（不懂是什么意思  
git branch (-m | -M) [<oldbranch>] <newbranch> 重命名分支，reflog 会做相应的重命名，并且创建条目记录该行为  
git remote add origin <url> 添加远程仓库 
git remote -v 查看远程仓库信息 
git remote rm origin 删除远程仓库

## 远程仓库 
git clone git@github.com:xxx cd xxx  git 支持多种协议但是 ssh 协议是最快的，
https 协议是慢的  git 的分支切换功能非常快，其他的项目管理软件非常慢 （1s内）  
git 中的无数 commit 组成一条时间线，一般项目的分支叫做 master，  HEAD 用于指向当前分支，而当前分支会指向指向 commit 链表 			

## 分支 
git branch -b dev 创建并切换分支（等价于以下两个命令） 
git branch dev 创建分支 
git checkout dev 切换分支  
git branch 查看分支  

# git 版本管理结构 	
    git 所管理的本质就是一个个用链表串联起来的 commit 	
    通过分支指向链表的某个末端 	
    再用 HEAD 指向当前分支 	 

git merge <branch> 将指定分支合并到当前分支（这里没有说怎么处理冲突！快进模式是非常快的，通常某个分支没有发生改变，就改改指针）  
git branch -d <branch> 删除分支，合并完成后删除  

使用 checkout 来切换分支和撤销修改，不太科学，实际上切换分支应该用 
    
    switch git switch -c dev 创建并切换到新分支 
    git switch master 切换到 master 分支
    
## 解决冲突 
当两个分支不在一个单链表上时（而是出现了分叉，像树枝一样），此时合并分支不再时快速模式，而是有可能出现冲突。 

git merge feature  或报错，提示分支冲突 

可以使用 git status 查看哪些文件冲突了！ 
直接查看冲突文件，会出现 
    
    <<<<<<<< HEAD 
    xxxxxxxxxx 
    ======== 
    yyyyyyyyyy 
    >>>>>>>> feature 

解决冲突后再 add commit 再 merge 即可！  
git log --graph --pretty=oneline --abbrev-comit 可以查看分支合并情况！

## 分支管理策略 
1. 合并时如果能用 FastForward 就会直接用，但是删除分支后会丢失信息 
2. 强制禁用 FastForward 模式 git 在 merge 时就会生成一个新的 commit 就可以从历史上看书分支信息  

这种强制的方法: git merge --no-ff

## 分支管理的基本原则 
1. 保证 master 分支的稳定性（仅用于发布版本，不在上边干活） 
2. dev 分支是不稳定的，在 dev 上干活

## stash 
**紧急 bug 修复** 
情景：当我正在 dv 上开发功能，开发到一半不能提交，但是此时收到一个紧急bug 
解决: 将当前的工作暂存，创建一个新的分支用于解决 bug，当处理完bug恢复之前未完成的工作区代码 
git stash 
git stash apply 
git stash pop  

使用 git cherry-pick <commit> 处理 bug 时可以拷贝分支


## 多人协作模式 
多人协作的工作模式通常是这样：   
首先，git push origin <branch-name>推送自己的修改；   
如果推送失败，则因为远程分支比你的本地更新，需要先用git pull试图合并；   
如果合并有冲突，则解决冲突，并在本地提交；   
没有冲突或者解决掉冲突后，再用git push origin <branch-name>推送就能成功！   
如果git pull提示no tracking information，则说明本地分支和远程分支的链接关系没有创建， 用命令git branch --set-upstream-to <branch-name> origin/<branch-name>。    

## rebase 变基 

主要功能是整理分支，通常如果push有冲突的话我们会先pull这时会在本地产生一个分支，并合并。 
再推送到远程 分支树就会很乱，所以应该在本地git rebase将本地修改移动到最近一次pull的版本后，这样分支就只有一个了！再提交就变得很优雅！  

## tag 
tag就是一个指向版本的指针，本来用id可以访问版本的，但是id不好看，所以可以用tag 

**使用方法** 

git tag v1.0 就在当前指向的commit打上标签了！ 
git tag  可以查看所有标签 
git tag <tag> <commit> 给历史提交打标签

标签是字母顺序而不是时间顺序 

git show <tag> 可以查看标签信息 

创建标签时 可以用
    
    -a 指定标签名 
    -m 添加说明 
    
git tag -d 删除本地标签，

git push origin <tag> 将标签推送到远程 
git push origin --tags 推送所有标签 

**删除远程标签**  
1. 删除本地标签 -d  	
2. 2. git push origin :refs/tags/<tag> 

删标签只记住这种方法就好了，其他方法存在误删分支的风险！ 冒号的原理是将 冒号前的推送到冒号后的， 这里冒号前是空的，就是将空的东西推送到远程分支！！！
