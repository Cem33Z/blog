---
title: "读书笔记-《Git版本控制管理（第二版）》"
date: 2017-10-07 20:49:53
tags: [Git, 读书笔记, Reading notes]
---

## 疑问 ##
1. 如果文件被多个人修改，commit上去合并时发生差异冲突，Git具体会怎么处理文件里的内容？

## 第三章 起步 ##

### git命令 ###

* 查看git版本：  `git --version`

* 获取git子命令列表：  `git help -a | git help -g`

* 创建初始版本库：
```
mkdir ~/public_html
cd ~/public_html
git init
```

* 将文件添加到版本库中
```
# 添加单个文件：
git add filename
# 添加当前目录及子目录下所有文件：
git add .
```

* 显示git当前状态： `git status`
* 配置提交者名字： `git config user.name "xdx"`
* 配置提交者邮箱： `git config user.email "xdx@off.gg"`
* 提交文件到版本库中
```
# 提交单个文件：
git commit filename
# 提交所有修改过的文件：
git commit -a
# 指定日志消息提交：
git commit -a -m "New edit"
```
* 查看所有git提交日志： `git log`
* 查看指定commit id提交日志
```
# 查看指定：
git show {commit id}
# 查看最新一条：
git show
```
* 查看当前开发分支简洁的单行摘要： `git show-branch --more=10`
* 查看指定commit id的提交差异： `git diff {commit id1}  {commit id1}`
* 从版本库中删除文件： `git rm filename`
* 从版本库里重命名文件： `git mv old-filename new-filename`
* 创建版本库副本： `git clone old-vcs new-vcs`

### git配置文件 ###
1. `.git/config`，版本库的特定配置设置，可以使用`--file`选项修改（默认选项），**拥有最高优先级** ；
```
git config user.name "xdx"
git config user.email "xdx@test-project.off.gg"
```
2. `~/.gitconfig`，用户级别配置设置，可以使用`--global`选项修改；
```
git config --global user.name "xdx"
git config --global user.email "xdx@off.gg"
```
3. `/etc/gitconfig`，系统级别配置设置，可以使用`--system`选项修改；
4. 列出在整组配置文件里共同查找的所有变量的配置：`git config -l`
5. 移除配置：`git config --unset --global user.email`
6. 设置命令别名

```
git config --global alias.show-graph 'log --graph --abbrev-commit --pretty=oneline'
git show-graph
```
## 第四章　基本的Git概念 ##

### 基本概念 ###

#### 版本库(repository) ####
 
1. 包含所有用来维护与管理项目修订版本和历史信息的数据库；
2. 除了提供版本库中所有文件的完整副本，还提供版本库本身的副本；
3. 把一个版本库克隆或复制到另一个版本库时配置设置不跟着转移；
4. 在版本库中，Git主要维护 **对象库和索引** 这两个主要数据结构；
5. 所有版本库数据存放在工作目录.git子目录里；

#### Git对象类型(object) ####

包含了原始数据文件和所有日志消息、作者信息、日期以及其他用来重建项目任意版本或分支的信息，是Git版本库实现的核心。对象库由以下4种原子对象构成：

* **块(blob=binary large object)**

文件的每一个版本表示为一个块，一个块保存一个文件的数据；

* **目录树(tree)**

一个目录树对象代表一层目录信息，记录了blob标识符、路径名和在一个目录里所有文件的一些元数据；

* **提交(commit)**

一个提交对象保存版本库中每一次变化的元数据（包括作者、提交者、提交日期和日志消息），每一个提交对象指向一个目录树对象；

* **标签(tag)**

给特定提交对象分配一个更友好的名字（如Ver-1.0-Alpha）；

为了更有效的利用磁盘空间和网络带宽，Git会把对象压缩并存储在打包文件(pack file)里，这些文件同样也在对象库中。

#### 索引(index) ####

* 索引是一个临时的、动态的二进制文件；
* 索引描述了整个版本库的目录结构，捕获项目在某个时刻整体结构的一个版本；
* 在提交前，索引会记录和保存通过Git命令执行的变更（如添加、删除或编辑文件的变更）；
* 索引中的变更是可以被修改的（删除或替换）；

#### 可寻址内容名称 ####

对象库中的每个对象都有一个唯一的名称，这个名称是由SHA1根据对象的完整内容，计算产生的一个SHA1散列值（也叫SHA1散列码、对象ID），是一种有效的全局唯一标识符。

#### Git追踪内容 ####

* Git追踪的是内容而不是文件名或目录名；
* Git根据文件内容计算每个文件的散列值后，将这个blob对象放到对象库里，并以SHA1散列值作为索引；
* 如果两个或多个文件的内容完全相同，Git在对象库里只保存一份blob形式的内容副本；
* 如果相同文件中某一个文件发生变化，Git会为它重新计算出一个新的SHA1散列值后，把这个新的blob加到对象库里，原来的blob在对象库里保持不变提供给没有变化的文件使用；
* 当文件版本发生变化时，Git的内部数据库会存储**每个文件的每个版本**（详细参考7.打包文件）

#### 路径名和内容 ####

Git只记录每个文件的路径名，确保能通过它的内容重建文件和目录。

#### 打包文件(pack file) ####

Git直接存储每个文件的每个版本，会不会太低效率了？否！Git使用了一种叫打包文件的存储机制，很好的解决了这个问题，具体流程如下：

1. Git先定位内容非常相似的全部文件，然后为其中之一存储整个内容；

2. 再计算相似文件之间的差异并且只存储差异；

3. 例如只是修改文件中的一行，Git可能会存储新版本的全部内容，然后记录那一行的更改作为差异，并存储在包里。

存储一个文件的整个版本并存储用来构造其他版本的相似文件的差异，这种机制其他VCS早已经用过，那么Git和他们的区别在哪？

由于Git是根据内容驱动的，所以它并不太关心两个文件之间的差异，是否属于同一个文件的不同版本。Git可以从版本库里任何地方取出两个文件并计算差异，只要它们足够相似并且能有很好的数据压缩效果。

### 对象库图示 ###

* blob对象是数据结构的“底端”，它只能被树对象引用；
* 树对象指向若干blob对象，或指向其他树对象。一个提交对象指向一个特定的树对象，而且这个树对象是由提交对象引入版本库的；
* 每个标签只能指向一个提交对象；
* 分支不是一个基本的Git对象，它在命名提交对象时起到关键作用；

### Git在工作时的概念（实例演示） ###
#### 进入.git目录 ####
* `.git/objects`目录用来存放所有Git对象
* `.git/index` 文件为索引

#### 对象、散列和blob ####

1. 通过blob的散列值从对象库中提取文件内容
```
git cat-file -p 3b18e512dba79e4c8300dd08aeb37f8e728b8dad
git cat-file -p 3b18e
```
2. 通过对象的唯一前缀查找对象的散列值：`git rev-parse 3b18e`

#### 文件和树 ####

Git通过目录树(tree)的对象来跟踪文件的路径名。当使用git add命令时，Git会给添加的每个文件的内容创建对象，但不会立即为树创建一个对象，而只是暂存在索引中。每次执行更改(git add,git rm,git mv)时，Git会用新的路径名和blob信息来更新索引。

从当前索引创建一个树对象：

1. 查看blob的散列值与文件的关联：`git ls-files -s`

2. 捕获索引状态并把它保存到一个树对象里：`git write-tree`

每次对相同的索引计算一个树对象时，产生的SHA1散列值仍是完全一样，Git不需要每次重新创建一个新的树对象。

3. 查看树对象：`git cat-file -p 68aba6`

#### 树层次结构 ####

```
cd /tmp/hello
mkdir subdir
cp hello.txt subdir/
git add subdir/hello.txt
git write-tree
git cat-file -p 4924
```

#### 提交 ####

```
echo -n "Commit a file that says hello\n"  | git commit-tree 4924
git cat-file -p 3b0c
find .git/objects/
```

默认情况下，作者(Author)和提交者(Commit)是同一个人（也有例外），可以使用`git show --pretty=fuller `  查看提交的其他细节

#### 标签 ####

Git有两种基本的标签类型：

* 轻量级(lightweight)，只是一个提交对象的引用，通常被版本库视为是私有的，而且这些标签并不在版本库里创建永久对象；

* 带附注(annotated)，会在版本库里创建一个对象，它包含提交信息并且可以进行数字签名；

创建一个带有提交信息、带附注且未签名的标签：

```
$ git tag -m "Tag version 1.0" V1.0
$ find .git/objects/
.git/objects/
.git/objects/pack
.git/objects/info
.git/objects/3b
.git/objects/3b/18e512dba79e4c8300dd08aeb37f8e728b8dad
.git/objects/3b/0ccc000def6f33d55e7ab44c0b506ff31f159f
.git/objects/68
.git/objects/68/aba62e560c0ebc3396e8ae9335232cd93a3f60
.git/objects/49
.git/objects/49/2413269336d21fac079d4a4672e55d5d2147ac
.git/objects/26
.git/objects/26/6fb2fd85ad8c2278142e733e99f70a1904cf83
$ git tag -m "Tag version 1.0" V1.0 3b0c
$ git rev-parse V1.0
4e94c57a9860c2d50406e9ecf4d88a8f435490ba
$ git cat-file -p 4e94
object 3b0ccc000def6f33d55e7ab44c0b506ff31f159f
type commit
tag V1.0
tagger xdx <xdx@localhost.localdomain> 1507297103 +0800
Tag version 1.0
```


## 第五章 文件管理和索引 ##
** 这里仅仅做一个测试而已 **
