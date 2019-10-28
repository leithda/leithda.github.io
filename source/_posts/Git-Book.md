---
title: Git-Book
categories:
  - Tools
  - Git
tags:
  - 版本控制
  - Git
author: 长歌
abbrlink: 2879265125
date: 2019-10-28 00:00:00
---

Git(读音为/gɪt/。)是一个开源的分布式版本控制系统，可以有效、高速地处理从很小到非常大的项目版本管理。 [1]  Git 是 Linus Torvalds 为了帮助管理 Linux 内核开发而开发的一个开放源码的版本控制软件。
本篇文章取自[Git - Book](https://git-scm.com/book/zh/v2)
<!-- More -->

## 获取git仓库

### 在现有目录初始化仓库
```bash
cd projectDir   # 进入到待初始化的仓库目录
git init
```

### 克隆一个现有仓库
- 使用 git 协议克隆仓库
```shell
git clone git@github.com:leithda/note.git
```

- 使用https 协议克隆仓库
```shell
git clone https://github.com/leithda/note.git
```

## 记录每次更新到仓库
### 检查当前文件状态
- 使用Git时，文件的生命周期如图: 从左到右依次为: 未追踪，未修改，修改，暂存区
{% asset_img git_file_stat.png Git文件的生命周期 %}
- 检查文件状态使用如下命令:
```bash
git status 
```
### 跟踪新文件
1. 新增文件(NewFunc.java),状态未 `untracked`
```bash
$ git status  
位于分支 master
未跟踪的文件:
  （使用 "git add <文件>..." 以包含要提交的内容）

    NewFunc.java

提交为空，但是存在尚未跟踪的文件（使用 "git add" 建立跟踪）
```
2. 使用`git add 参数`命令跟踪一个新的文件，其中的参数可以是单个文件也可以是指定文件夹,添加后的文件状态为 `staged`
```bash
# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master x [17:23:35] 
$ git add . # 这里使用 . 表示当前文件夹

# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master x [17:24:32] 
$ git status .
位于分支 master
要提交的变更：
  （使用 "git reset HEAD <文件>..." 以取消暂存）

    新文件：   NewFunc.java

```
- 此处可以使用`git diff --cached`命令查看文件变动，由于是新文件，Git中不曾保存这个文件的快照，所以输出为空。

3. 使用`git commit`命令提交暂存区文件到本地git仓库，推送后的文件状态为 `unmodified`
```bash
# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master x [17:25:45] C:130
$ git commit -m "提交信息:新增NewFunc.java 增加了新功能sayHi，具体方法说明见借口文档"
[master 8b0f16d] 提交信息:新增NewFunc.java 增加了新功能sayHi，具体方法说明见借口文档
 1 file changed, 6 insertions(+)
 create mode 100644 NewFunc.java
```
- 此时查看当前文件状态，没有需要追踪及提交的文件
```bash
# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master o [17:29:02] 
$ git status NewFunc.java 
位于分支 master
无文件要提交，干净的工作区
```

### 暂存已修改文件
1. 修改原有文件`Main.java`
```java
public class Main{

        public static void main(String[] args){
                // System.out.println("hello world");
                NewFunc.sayHi("sunline");
        }
}
```

2. 查看文件状态
```bash
# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master x [17:34:26]
$ git status .
位于分支 master
尚未暂存以备提交的变更：
  （使用 "git add <文件>..." 更新要提交的内容）
  （使用 "git checkout -- <文件>..." 丢弃工作区的改动）

    修改：     Main.java

修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）
# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master x [17:34:28] 
$ git add Main.java 

# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master x [17:37:21] 
$ git status .
位于分支 master
要提交的变更：
  （使用 "git reset HEAD <文件>..." 以取消暂存）

    修改：     Main.java



```
- 其中，`git add <文件>`，是将文件修改加入暂存区(staged)
- `git checkout -- <文件>`会将文件恢复到Git保存的最后一版快照时的状态
- 此时使用`git diff`查看文件前后改变
```bash
diff --git a/Main.java b/Main.java
index 997489c..d7660c3 100644
--- a/Main.java
+++ b/Main.java
@@ -1,6 +1,7 @@
 public class Main{
 
        public static void main(String[] args){
-               System.out.println("hello world");
+               // System.out.println("hello world");
+               NewFunc.sayHi("sunline");
        }
 }
```
- 明显看出，删除了`System.out.println("hello world");`，增加了`// System.out.println("hello world");`以及`NewFunc.sayHi("sunline");`

3. `git commit `提交，不再赘述

## 查看提交历史
- 查看提交历史使用`git log`命令

### 查看提交
```bash
# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master x [19:37:07] 
$ git log -2  

commit 8b0f16de407b412cd5a9cc3d0acab8d4ff7ef8d4 (HEAD -> master)
Author: leithda <leithda@163.com>
Date:   Mon Oct 28 17:29:02 2019 +0800

    提交信息:新增NewFunc.java 增加了新功能sayHi，具体方法说明见借口文档

commit a9e20b7581df601e7120683b356a5339dfc81cd8
Author: leithda <leithda@163.com>
Date:   Mon Oct 28 17:13:02 2019 +0800

    初始化仓库
commit 8b0f16de407b412cd5a9cc3d0acab8d4ff7ef8d4 (HEAD -> master)
Author: leithda <leithda@163.com>
Date:   Mon Oct 28 17:29:02 2019 +0800

    提交信息:新增NewFunc.java 增加了新功能sayHi，具体方法说明见借口文档

commit a9e20b7581df601e7120683b356a5339dfc81cd8
Author: leithda <leithda@163.com>
Date:   Mon Oct 28 17:13:02 2019 +0800

    初始化仓库
```
- 其中-2用于表示查看最后两次提交记录，

### 查看提交历史及更改内容
- 加入 `-p 参数`,输入`git log -p -1`,查看最后一次`commit`内容
```bash
commit 8b0f16de407b412cd5a9cc3d0acab8d4ff7ef8d4 (HEAD -> master)
Author: leithda <leithda@163.com>
Date:   Mon Oct 28 17:29:02 2019 +0800

    提交信息:新增NewFunc.java 增加了新功能sayHi，具体方法说明见借口文档

diff --git a/NewFunc.java b/NewFunc.java
new file mode 100644
index 0000000..df9d81a
--- /dev/null
+++ b/NewFunc.java
@@ -0,0 +1,6 @@
+
+public class NewFunc{
+       public static void sayHi(String name){
+               System.out.println("hi, "+name+" . today is nice day!");
+       }
+}
```

### git log 的常用选项

| 选项            | 说明                                         |
| --------------- | -------------------------------------------- |
| -p              | 按补丁格式显示每个更新间的差异               |
| `--stat`        | 显示每次更新的文件修改统计信息。             |
| `--shortstat`   | 只显示 --stat 中最后的行数修改添加移除统计。 |
| `--name-only`   | 仅在提交信息后显示已修改的文件清单。         |
| `--name-status` | 显示新增、修改、删除的文件清单。             |
| `--graph`       | 显示 ASCII 图形表示的分支合并历史。          |


## 撤销操作
### 漏提文件

- 提交`Part1.java`后发现漏提了`Part2.java`,此时执行`git add Part2.java`然后`git commit --amend`,输入界面如下:

```bash
提交part1,漏提Part2，补充提交

# 请为您的变更输入提交说明。以 '#' 开始的行将被忽略，而一个空的提交
# 说明将会终止提交。
#
# 日期：  Mon Oct 28 21:17:14 2019 +0800
#
# 位于分支 master
# 要提交的变更：
#       修改：     Main.java
        新文件：   Part1.java
        新文件：   Part2.java
#

```

- `git log`查看

```bash
commit 6c00166e11bbebef2c82fbbec8734869d1fcc38b (HEAD -> master)
Author: leithda <leithda@163.com>
Date:   Mon Oct 28 21:17:14 2019 +0800

    提交part1	# 原提交信息
    
    提交part1,漏提Part2，补充提交	# 补充提交信息
    
            新文件：   Part1.java
            新文件：   Part2.java


```

### 取消暂存的文件

- 此时有新增(或修改)的两个文件，两次内容应该分开提交，执行`git add *`命令都添加到了暂存区，作如下操作取消文件追踪.

```bash
# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master x [21:28:07] 
$ git status .
位于分支 master
要提交的变更：
  （使用 "git reset HEAD <文件>..." 以取消暂存）

	新文件：   CommitFirst.txt
	新文件：   CommitSecond.txt


# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master x [21:28:09] 
$ git reset HEAD CommitSecond.txt # 取消CommitSecond.txt文件的追踪

# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master x [21:28:20] 
$ git status . # 此时提交的话，只有CommitFirst.txt文件产生快照，存入仓库                  
位于分支 master
要提交的变更：
  （使用 "git reset HEAD <文件>..." 以取消暂存）

	新文件：   CommitFirst.txt

未跟踪的文件:
  （使用 "git add <文件>..." 以包含要提交的内容）

	CommitSecond.txt

```

### 撤销对文件的操作

- 此时修改文件`Main.java`，修改后状态及取消操作如下：

```bash
git diff
==============git diff 内容如下，代码中增加了注释=============
diff --git a/Main.java b/Main.java
index d7660c3..aa68318 100644
--- a/Main.java
+++ b/Main.java
@@ -2,6 +2,6 @@ public class Main{
 
        public static void main(String[] args){
                // System.out.println("hello world");
-               NewFunc.sayHi("sunline");
+               NewFunc.sayHi("sunline");       // 调用方法sayHi
        }
 }
================ git diff 结束 ============================

# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master x [21:33:49] 
$ git status .
位于分支 master
尚未暂存以备提交的变更：
  （使用 "git add <文件>..." 更新要提交的内容）
  （使用 "git checkout -- <文件>..." 丢弃工作区的改动）

	修改：     Main.java

修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）

# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master x [21:34:48] 
$ git checkout Main.java  # 执行这个命令后，文件内容恢复到修改前版本
```

## 远程仓库的使用
### 查看远程仓库
```bash
$ git remote -v
gitee	git@gitee.com:leithda/leithda.git (fetch)
gitee	git@gitee.com:leithda/leithda.git (push)
origin	git@github.com:leithda/leithda.github.io.git (fetch)
origin	git@github.com:leithda/leithda.github.io.git (push)
```

- 其中，前面是对应仓库的简写，后面为仓库对应的url地址.从地址中可以看出，仓库使用的是`git`协议

### 远程仓库的添加

- 添加远程仓库的命令如下: `git remote add <shortname> <url>`,其中 `shortname`是简写，如刚才的`gitee`

```bash
$ git remote add gitee git@gitee.com:leithda/leithda.git
```



### 推送到远程仓库

- 对应命令为 `git push [remote-name] [branch-name]`，推送到`origin`远程仓库的`master`分支使用如下命令。这里的`origin`可以替换为`gitee`推送到另外一个远程仓库中。

```bash
git push origin master
```

## 分支管理

------

### 查看分支

```bash
$ git branch -a
#=======================
  draw_lsd
  m1
  m2
* master
  transfer_bc
```

### 新建分支

- 举个例子，我们想解决一个关于取款的问题，决定新建一个`draw_lsd`分支供我来解决问题。仍使用刚才的git仓库举例

```bash
$ git branch draw_lsd

# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master o [21:59:53] 
$ git checkout draw_lsd 
切换到分支 'draw_lsd'
```

- 以上两条命令可以简写为`git checkout -b draw_lsd`
- 假设我需要修改`Part1.java`及`Part2.java`

```bash
$ git log 
commit 64fad3bb26c612b6e72012e3b31f070e0d3b8839 (HEAD -> draw_lsd)
Author: leithda <leithda@163.com>
Date:   Mon Oct 28 22:03:42 2019 +0800

    解决取款问题 by lsd

```
  - 修改文件并提交`git add`，`git commit`，使用`git log`查看。

### 合并分支

#### 主分支直接合并子分支

- 此时切换回主分支，`git checkout master`,创建白晨解决问题分支`git checkout -b transfer_bc`,(**备用**),此时分支结构如下:

```bash
  draw_lsd
* master #(*指向当前分支)
  transfer_bc

```



- 合并刚才我修改的分支

```bash
$ git merge draw_lsd 
更新 f00bc18..64fad3b
Fast-forward
 Part1.java | 2 ++
 Part2.java | 2 ++
 2 files changed, 4 insertions(+)
```

- 没有冲突，我刚才修改的文件已经完美的合并到了主分支`master`上.注意此时主分支的基

#### 子分支检查冲突后，合并到主分支

1. **非冲突情况**。此时，切换到小白修改分支，`git checkout transfer_bc`,首先修改`Transfer 第一个问题`，修改`Main.java`文件，此时不会产生冲突，合并流程如下:

```bash
$ git merge master
# 合并后git log 查看记录
$ git log 
commit c40f244cc171e530cf92e7916baf0acbe2eef293 (HEAD -> transfer_bc)
Merge: 68d0958 64fad3b
Author: leithda <leithda@163.com>
Date:   Mon Oct 28 22:29:35 2019 +0800

    Merge branch 'master' into transfer_bc

commit 68d095865b045b409ef774f6b3c5ced2418ae5a5
Author: leithda <leithda@163.com>
Date:   Mon Oct 28 22:29:22 2019 +0800

    解决第一个转账问题 by bc


# 多了一条合并记录
```

- 此时切换回主分支，合并当前第一次修改

```bash
$ git checkout master                     
切换到分支 'master'

# leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master o [22:35:41] 
$ git merge transfer_bc 
更新 64fad3b..c40f244
Fast-forward
 Main.java | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

2. **冲突情况**，创建分支`m1`和分支`m2`，修改`m1`分支文件`NewFunc.java`

   - m1合并主分支检查冲突,无冲突，合并到主分支

     ```bash
     $ git merge master              
     已经是最新的。	# 表明从主分支创建当前分支以来，主分支内容没有变动
     
     # 直接切换主分支进行合并
     $ git checkout master           
     切换到分支 'master'
     
     # leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master o [22:43:52] 
     $ git merge m1       
     更新 c40f244..9978790
     Fast-forward
      NewFunc.java | 9 +++++++++
      1 file changed, 9 insertions(+)
     ```

   - m2修改文件并执行合并操作

     ```bash
     # 省略修改文件，add,commit操作
     $ git merge master              
     自动合并 NewFunc.java
     冲突（内容）：合并冲突于 NewFunc.java
     自动合并失败，修正冲突然后提交修正的结果。
     
     $ git status .    
     位于分支 m2
     您有尚未合并的路径。
       （解决冲突并运行 "git commit"）
       （使用 "git merge --abort" 终止合并）
     
     未合并的路径：
       （使用 "git add <文件>..." 标记解决方案）
     
     	双方修改：   NewFunc.java
     
     修改尚未加入提交（使用 "git add" 和/或 "git commit -a"）
     ```

   - 解决冲突，此时文件内容如下

     ```java
     public class NewFunc{
     	/**
     	 * 老板说此处应该有注释
     	 * add by m2
     	 */
     	public static void sayHi(String name){
     		System.out.println("hi, "+name+" . today is nice day!");
     	}
     <<<<<<< HEAD
     	
     	/**
     	 * add by m2
     	 */
     	public static void m2(String name){
     		System.err.println("Error : "+name+"today is a bad day!!");
     =======
     
     	/**
     	 * 老板说，hi不好听，非让改成hello
     	 * 顺便修改了下语法错误.
     	 * modify by m1 on 2019年10月28日
     	 */
     	public static void m1(String name){
     		System.out.println("hello, "+name+". today is a nice day!");
     >>>>>>> master
     	}
     }
     ```

     - 其中`<<<<<<< HEAD`到`=======`之间的内容为m2(我修改的存在冲突的内容)。`=======`至`>>>>>>> master`为合并分支(master)中存在冲突的内容
     - 修改文件内容如下，解决冲突

     ```java
     public class NewFunc{
     	/**
     	 * 老板说此处应该有注释
     	 * add by m2
     	 */
     	public static void sayHi(String name){
     		System.out.println("hi, "+name+" . today is nice day!");
     	}
     
     	/**
     	 * 老板说，hi不好听，非让改成hello
     	 * 顺便修改了下语法错误.
     	 * modify by m1 on 2019年10月28日
     	 */
     	public static void m1(String name){
     		System.out.println("hello, "+name+". today is a nice day!");
     
     	/**
     	 * add by m2
     	 */
     	public static void m2(String name){
     		System.err.println("Error : "+name+"today is a bad day!!");
     	}
     }
     ```
     - 执行提交两部曲,`git add`、`git commit`,切换主分支合并当前分支(注，此时还应先检查是否存在冲突，此处未避免冗余省略)

       ```bash
       # leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:m2 x [23:05:49] 
       $ git add .         
       
       # leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:m2 x [23:05:56] 
       $ git commit -m "解决问题 解决冲突 by m2"
       [m2 e7eb9fe] 解决问题 解决冲突 by m2
       
       # leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:m2 o [23:06:09] 
       $ git checkout master                    
       切换到分支 'master'
       
       # leithda @ leithda-GL502VML in ~/work/tmp/testGit on git:master o [23:06:21] 
       $ git merge m2       
       更新 9978790..e7eb9fe
       Fast-forward
        NewFunc.java | 10 ++++++++++
        1 file changed, 10 insertions(+)
        
       $ git log 
       commit e7eb9fe40b752f96cfdceaf6065206bf8c9453a4 (HEAD -> master, m2)
       Merge: 4290aca 9978790
       Author: leithda <leithda@163.com>
       Date:   Mon Oct 28 23:06:09 2019 +0800
       
           解决问题 解决冲突 by m2
       
       commit 4290aca1bbf280ccac4abf75a03c602e3ee4e79f
       Author: leithda <leithda@163.com>
       Date:   Mon Oct 28 22:58:03 2019 +0800
       
           解决问题 by m2
       
       commit 997879005a061df2dd31c1144ed774ec4a3b32b2 (m1)
       Author: leithda <leithda@163.com>
       Date:   Mon Oct 28 22:42:21 2019 +0800
       
           解决问题 by m1
       
       commit c40f244cc171e530cf92e7916baf0acbe2eef293 (transfer_bc)
       Merge: 68d0958 64fad3b
       Author: leithda <leithda@163.com>
       Date:   Mon Oct 28 22:29:35 2019 +0800
       
       
       ```
       
       - 关于先使用子分支去`merge`主分支的优点，对冲突的解决放在子分支内，不会影响主分支上的代码，避免更多的冲突产生。缺点是步骤过于繁琐。

### 删除分支

#### 删除合并后的分支

```bash
$ git branch -d [branch-name]
```

#### 删除未合并的分支

```bash
$ git branch -D [branch-name]
```

