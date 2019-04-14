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

* 
* 













