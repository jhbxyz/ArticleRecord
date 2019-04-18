# git 命令

* git --version
* pwd 查看当前工作路径
* ls -al 看文件
* ls 看文件
* clear 清屏
* cp ../material/Readme.md .  拷贝同级文件夹下的文件
* cp ../material/Readme.md  readme  拷贝同级文件夹下的文件 **并修改名称**
* cp -r ../material/image .  拷贝同级文件夹下的文件夹
* mv Readme.md  readmesss.md  修改文件名( Readme.md-> readmesss.md )
* mkdir 文件名 (创建文件夹)
* echo "hello,world" > readme  (创建文件 在readme文件中写入hellow,world)

### 配置 user 信息

- git config --global user.name 'your name'

- git config --global user.email 'your_email@domain.com'

-  ====config 的三个作用域 **(缺省等于local)**=======

- git config --local  只对仓库有效

- git config --global  对登录用户所有仓库有效

- git config --system  对系统所有用户有效 (不常用)

- =======显示 config 配置=======

- git config --local --list

- git config --global --list

- git config --system --list

- =======清除 config 配置=======

- git config --unset --local user.name

- git config --unset --global user.name

- git config --unset --system user.name

- ==========**优先级 local>global>system**==========

  

- git config --list 看配置

  

### 建 git仓库

##### 已有项目(git 管理)

* cd 到项目路径

* git init 

##### 新建项目(git 管理)

* cd 到指定文件夹

* git init your_project

* cd your_project

* ====================

* 

* git status (查看当前状态)

* git add 具体文件名 (用git 管理该文件)

* git commit -m  "日志内容" (提交到本地仓库)

* git log (查看 log)

* git log --oneline (只看log 的一行注释)

* git log -n2 --oneline (只看最近的两条)

* git add-u  (更新 暂存区已经提交过的用次命令会直接更行并加入到暂存区)

* git rm '文件名'  ( 删除文件 对应的是 **git add '文件名'**)

* git reset  --hard (**危险命令 会清除暂存区的内容,类似于所有工作区更改的文件revert**)

  ====重命名文件====

  * 方法一 :(需要三步)
    * mv  原文件名   新文件名
    * git add '新文件名'
    * git rm ' 原文件名'

  * 方法二 :
    * git mv 原文件名  新文件名

* git branch -v

* git checkout -b temp  commit的版本号 (以当前的commit 来切换一个temp分支)

* git help --web log  (通过网页看log 的用法)

* gitk  (图形化界面 管理git )

* git cat-file -t  commit的版本号  ( 查看文件类型)

* git cat-file -t  commit的版本号  ( 查看文件内容)

### commit tree blob 之间的关系

* commit 是tree (树就是文件夹) 是当时commit时的所有文件夹下的内容
* blob相当于是文件




* git diff HEAD HEAD^1  (比较最近的两次提交的差异)
* git branch -d 分支名 (删除分支)
* git branch -D 分支名 (删除分支(**在风险可控的情况下用**))
* git commit --amend  修改**最近一次**提交的log message
* git rebase -i  本地commit 历史 提交 变更message

* git diff --cached (比较暂存区和HEAD(commit)的区别)
* git diff  (比较工作区和暂存的的区别)
* git reset HEAD (让暂存区恢复成和HEAD一样).
* ===变工作区用checkout   变暂存区用reset===
* git checkout -- 文件名 (把工作区的内容恢复到和暂存区一致)
* git reset HEAD --文件名 (把暂存区某一个文件恢复到上一次commit一致)



* git reset --hard commit的id  (恢复)
* git diff  分支1  分支2   文件名 (比较两个分支此文件的差异) ==> git diff commitId号  commitId号  也同样效果
* rm 文件名 (删除此文件)
* git rm 文件名 (把删除的文件添加到暂存区)
* git reset --hard HADE (把工作区和暂存区恢复成和头指针一直(也就是最后一次commit)



* git stash (临时存贮) 
* git stash list
* git stash apply (可以恢复原来文件 但是 git stash list 里面的对象一直存在 你可以反复使用)
* git stash pop (恢复原来文件, 并删除存储的对象)



* rm -rf 文件名 (删除文件)



* git remote -v (查看远端仓库)
* git  reomte add 仓库名 路径 (和远端仓库关联)
* git push --set-upstream  本地仓库名  远端仓库名 (和远端仓库关联)



# 和远端服务器玩儿



* cd ~/.ssh  (看是否有ssh 文件)
* ssh-keygen -t rsa -b 4096 -C "your_simple@email.com"
* git  remote add 名称  ssh地址 (增加远端)
* git push 远端名称 --all (把本地所有分支 push 到远端)
* git fetch 
* git merge
* git push
* git cheout -b 本地分支名  远端分支名 (本地分支和远端分支关联)
* 























