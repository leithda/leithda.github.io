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

