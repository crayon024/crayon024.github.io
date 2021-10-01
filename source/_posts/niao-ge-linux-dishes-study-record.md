---
title: 《鸟哥的Linux私房菜》学习记录
date: 2021-05-22 18:54:14
updated: 2021-05-22 18:54:14
categories: Linux
tags: 
  - linux
---

硬件 - 内核 - 系统调用 - 应用程序。操作系统（Linux 系统）就在 内核 - 系统调用那两层。

<!--more-->

因为不同的硬件提供的功能函数不同，所以一个操作系统可以在 Intel 的 x86 架构的硬件平台上运行，但是无法在采用其他架构的硬件上运行。早期苹果公司在 IBM 的 PowerPC CPU 硬件架构上发展的 Mac 电脑，就无法运行 Windows 系统（基于 Intel x86 架构开发）。

每种操作系统都是在针对特定的硬件平台上运行的。不过因为 Linux 是开源的，即大家可以获取到它的源代码，就可以在此基础上针对不同的硬件平台修改代码来运行。

## 磁盘分区

[挂载](https://en.wikipedia.org/wiki/Mount_(computing)#:~:text=Mounting%20is%20a%20process%20by,via%20the%20computer's%20file%20system.)**，**文件系统和存储设备的关系。挂载就是指将文件系统中的目录**挂载**在存储设备的某个位置上，用户访问这个目录下的文件时，操作系统就会该目录对应的**挂载点**读取文件。一般这个进入的目录也称为挂载点。

Linux 中根目录的重要性不言而喻，所有根目录一定是挂载到某个分区的。其他的目录用户可以根据自己的需求挂载到不同的分区。

如果 / 目录挂载在分区 1，home 目录挂载在分区 2，那么 /test/home/myfile/two，那么 two 这个文件是在 home 所在的分区还是 根目录所在的分区呢？**通过反向查找挂载点即可，先找到的挂载点在哪就是哪个挂载点。**这里 two 使用的就是 /home 这个挂载点下对应的分区进行存储。

## 命令行模式下的一些基础概念

基础格式：

```plain
$ command [-options] parameter1 paremeter2 ...
$ ll -al ../my
$ ll -a -l ../my
$ ll ../my
```

Shift + PageUP 或 PageDown 在**命令行模式下**进行翻页。要是没有 PageUP 和 Down 键怎么办？

比如执行 cat 在命令行输出满屏了，可以通过**管道 |**把输出结果给可以翻页的命令，比如 less 或者 more。就可以通过 b 或者 空格键快速上下翻页了。

```plain
$ cat fullfile | less
```

```plain
$ command --help，命令的简单说明，比如有哪些参数可用。
```

```plain
$ man command，命令的详细操作手册。
```

执行 man 后，会进入一个类似 Vim 的界面，可用通过 PageUP 或 PageDown（或者空格键 ）进行翻页。/String，或者 ？String，向下 向上查找出现了 String 的内容。并通过 n/N 查找匹配的下一个/上一个。
巨简单的一种文本编辑器：nano。图片底部的 ^ + 字母表示，Ctrl + 字母就会执行对应的操作。

![image (1)](niao-ge-linux-dishes-study-record/image (1).png)

## 文件属性管理

### chgrp，chown，chmod

```plain
修改文件所属用户组， -R 递归操作所有子目录
$ chgrp [-R] groupName filename/dirname

修改文件拥有者，-R 递归操作所有子目录
$ chown [-R] useName filename/dirname

修改文件权限
1）数字形式： read = 4 = 2^2，write = 2 = 2^1，x = 1 = 2^0
$ chmod [-R] 761 filename/dirname
2）符号形式：a - 全部用户，o - ohters，u - user，g - group
$ chmod [-R] u+rwx,g=rx,o-rwx,a+rwx filename/dirname
```

**权限对于文件和目录的不同作用意义**：

对于文件，指对**文件内容**的操作权限。要注意的是对文件的 write 权限，并不具备删除该文件的功能。

对于目录，read 权限表示可以读取目录结构列表的能力。w 则是可以删除，新增文件等等。**对于目录的 x（执行权限）则代表的是用户是否有进入该目录的能力**。

## Linux 目录配置

Linux 的世界中所有东西的抽象为文件。Linux 的目录配置指的是各个不同版本的 Linux 各种目录大致应该存放什么文件，因此也诞生了 FHS（Filesystem Hierarchy Standard） 标准。

## 目录管理

### cd，pwd，mkdir

```plain
切换到当前用户的 home 目录
$ cd ~

回到上一次的工作目录
$ cd -

$ pwd [-P], -P 选项执行输出真正路径，而非链接路径（对链接文件来说有用）

-p 选项，直接创建多级目录
$ mkdir -p my1/my2/my3
```

使用 **mkdir** 创建的新目录默认权限是什么呢？这和 umask 有关。不过你可以也在 **mkdir** 时使用 **-m 777** 来指定权限。

### cp，rm，mv

* cp [源文件] [目标文件]
  * -a，一般来说 cp 复制后的文件拥有者一般是操作命令者本身。添加 -a 选项就可完完全全复制文件属性，包括权限，创建时间等等。
  * -r，目录的话可能你需要递归复制
* rm 文件或者目录
  * -f，强制删除
  * -r，删除目录需要进行递归删除
* mv

## 文件管理

### less，cat，head，tail，od，touch

* head [-n number] filename
* less，和 man 命令执行后的操作很像，比如 空格 对应 Page Down，b 对应 Page Up 等等
* od，查看非文本文件
* touch，新建文件或者修改文件时间
  * atime，access time
  * mtime，modify time
  * ctime，status time，比如文件权限改变的时间。

### 文件和目录的默认权限：umask -S，umask 新的 umask 值

![image (2)](niao-ge-linux-dishes-study-record/image (2).png)

0022 的数字指的是该默认权限需要减掉的权限。第一位的 0 个人猜测是 root 用户的，似乎没办法改变。后三位 022 的 0 代表 u = rwx，2 代表 group-w（即 g = rx）,同理最后一位的 2 一样，只是作用的用户是 others。

### 查找文件

* locate regexWord，从已建的数据库中查询，所以不用到处查磁盘。但是数据库更新频率不高，CentOS 7 是一天一更。可以使用 updatedb 命令更新，这个命令会花一些时间。
  * locate /etc/sh
  * locate  ~/m
  * locate -i ~/m
* find **[PATH]**
  * find ./ -ctime 4，**当前目录下**，4 天前的那一天修改过 status 的文件
  * find ./ -mtime -4，4 天内被修改过内容的文件
  * find ./ -mtime +5，5 天包括更久之前修改过内容的文件
* find . -name 
  * [Linux的五个查找命令 - 阮一峰的网络日志 (ruanyifeng.com)](http://www.ruanyifeng.com/blog/2009/10/5_ways_to_search_for_files_using_the_terminal.html)
* whereis
* which

## 

## 文件系统

### 磁盘和目录的容量：df，du

* df，列出文件系统整体的磁盘使用情况
  * -h，以我们易理解的方式输出。比如多少 G，多少 M
* du [options] [文件名称或者目录名称]

### 硬链接和符号链接：ln

在 CentOS 7.x 后，默认文件系统采用 xfs 系统。

有一些文件相关的特点需要了解：

* 每个文件占用一个 inode，**文件内容**由 inode 记录来指向。
* 想要读取文件内容，需要正确的 inode 号码才能进行读取。

**硬链接**在某个目录下新增一个文件名并链接到某个 inode 号码指向的内容。也就是说同一处的文件内容可以通过不同的文件名来进行操作。和符号链接（软链接）不同的点在于硬链接删除了其中任何一个文件，其实 inode 是还在的。

**符号链接**则是在某个目录下新建一个文件名指向某个文件，这个文件名的虚的，只起到一个引用的作用。

* ln 源头文件 新建链接文件
  * -s，添加个该选择设置符号链接，不添加默认硬链接

## 

### 文件的压缩

压缩文件我们非常常见，一般我们可以通过文件后缀名区分文件是否被压缩且使用的压缩技术。比如 .zip，.gzip，.tar.gzip 等等。Linux 不像 Windows 通过文件后缀名辨别各种文件类型，比如 .exe，.txt，.mp3，.doc  。还记得 ll 命令或者 ls 命令的输出结果，其中第一个字符才表示对应的文件类型，- 表示普通文件，d 表示文件夹，l 表示链接文件等等。所以在 Linux 中文件后缀名对文件类型是没有什么意义的，但是有时候我们可以通过合适的文件后缀名来清晰文件类型。压缩文件也是如此。

简单理解一下压缩原理，操作系统通过机器码存储文件，比如 1000 0000 ，压缩技术类似将 1000 0000 处理为1 0*8 的方式处理并存储，解压缩的时候规则将实际的机器码复原即可。

### Linux 中常见的压缩命令

* .zip，zip 程序压缩
* .gz，gzip 程序压缩
* .tar.gz，tar 程序打包的文件通过 gzip 压缩
* .tar.bz2，tar 程序打包的文件通过 bzip2 程序压缩

通过 压缩命令仅对一个文件进行压缩解压缩，所以通过 tar 将多个文件打包为一个文件，在通过压缩命令来提高效率。

#### **gzip**

运行 gzip 产生的文件后缀为 .gz，当你使用 gzip 压缩文件的时候，源文件会被压缩为 .gz 文件，就是说源文件不存在了（这和 Windows 上很不一样）。

* gzip
  * -c，**压缩，**并把压缩的数据输出到屏幕上。**可以配合数据重定向到压缩文件并保留源文件。**
  * -d，**解压缩**
  * -t，检验压缩的一致性。-t filename1 filename2
  * -v，显示压缩信息

```plain
gzip -c mytxt.txt > mytxt.gz
```

cat，less，more 读取未压缩的纯文本文件，对应的可以使用 zcat，zless，zmore 读取。还有 zgrep，等等。

#### bzip2，xz

用法是 gzip 大致相同，生成的后缀名为 .bz2，且 bzip2 的压缩率比较高，但是花费的时间可能会更多一些。

xz 生成的压缩文件后缀名 .xz，压缩率更高，时间可能更久些。

### 打包命令：tar

虽然 gzip 也可以针对目录使用，添加 -r 选项即可，不过作用是**对目录中的文件分别进行压缩。**这时候可以用 tar 命令将多个文件打包，再进行压缩。

* tar（-c,t,x 。-z,j,J 不同时出现在一个命令行中）
  * -c，建立打包文件(tar 文件？)，可搭配 -v
  * -t，查看打包文件(tar 文件？) 中含有哪些文件名
  * -x，解包或者解压缩文件，可以搭配 -C ，把文件解压到特定的目录
    * -z，通过 gzip 支持压缩/解压缩，最好把后缀命名成 .tar.gz
    * -j，通过 bzip2 支持压缩/解压缩，.tar.bz2
    * -J，通过 xz 支持压缩解压缩，.tar.xz
      * -v，过程中显示正在处理的文件名
      * -f，后紧跟处理的文件名
        * -p
        * -P

#### **实战**：

* 打包压缩：tar -zcv -f filename.tar.gz 要被压缩的文件或目录名
* 查询：tar -ztv -f filename.tar.gz
* 查询：tar -jtv -f filename.tar.bz2
  * 234ASD在：tar -xjv -f filename.tar.xz -C 指定的在哪个目录解压

### 其他常见的压缩和备份工具：dd，cpio

## vim 和 Shell

### vim 的缓存，恢复和重新打开时的警告信息

使用 vim 编辑文件时，vim 会在编辑文件的同个目录下建立一个 .**原文件名.swp**的文件保存你对原文件的操作记录。这样可以在意外的情况下恢复你上次可能未保存编辑的操作。

因为 vim 被异常结束，导致交换文件没有按照正常流程结束，所以交换文件会保留下来。

![image (3)](niao-ge-linux-dishes-study-record/image (3)-2561646.png)

当你重新打开文件的时候，会提示你存在交换文件，你可以选择最后一行提供的 6 种操作。

![image (4)](niao-ge-linux-dishes-study-record/image (4).png)

* E，不加载交换文件的内容直接编辑。
* R，从交换文件恢复操作，但是交换文件还是存在目录中，可以手动删除避免每次打开出现类似提示。

### 数据重定向

一般执行一个命令的时候，从文件读取数据，通过标准输出/标准错误输出到屏幕中。命令正确执行通过标准输出，错误通过标准错误输出。且有对应的代码表示：

* 1，默认表示标准输出。
* 2，表示标准错误输出。
* 0，默认表示标准输入。

![image (5)](niao-ge-linux-dishes-study-record/image (5).png)

有了对应的代码后，我们可以通过对应的信息将本应该**输出**到屏幕中的内容重定向（>,>>）到文件中。

实战：将正确和错误结果分别重定向到不同文件中

```plain
$ find /ect/test -name fhx.txt > writePut 2> wrongPut
```

那如何将正确和错误的结果输入到同一个文件中？

```plain
$ find /etc/test -name fhx.txt > list 2> list
```

上面的做法理论上是对的，但是 list 文件可能会很混乱，因为无法保证正确和错误按照顺序写到文件中。应该这样做：

```plain
$ find /etc/test -name fhx.txt > list 2>&1
```

**标准输入 “<”，就是将原本本该由键盘输入获得的内容改为从文件来获取**。

“<<”，表示进行结束操作的输入字符。<< “stttop”,从键盘获得了 stttop 输入后就会停止输入操作。

## 进程

一个**程序被加载到内存**中运行，在**内存中**的那部分数据就被称为一个进程。在 Linux 中，所有的东西都被视为文件，但我们执行一个命令的时候，其实就是在运行其中的某一个文件。

比如我们执行 bash 命令，其实是将 /bin/bash 这个文件加载到内存运行。这部分数据就称为为一个进程，**Linux 会为其分配一个 PID（process id）,同时根据执行该进程的用户的相关属性，赋予该进程一组相关的权限设置（UID/GID）。**

执行 bash 命令后，相当于为用户新建了一个交互的 shell，我们在这个 shell 下执行其他命令时，产生的新进程其实是衍生自 bash 命令产生的进程。**由一个进程衍生出来的其他进程，在一般状态下会沿用父进程的相关权限属性。**可以执行 ps -l，观察 PPID（parent PID）了解进程的父进程。

Linux 的程序调用流程通常是 fork and exec，由父进程复制一个完全相同的子进程（PID 不同），然后 exec 执行实际要执行的进程。

### 任务管理：&（后台执行），ctrl+z（后台暂停），fg，bg

* **&**，在你要执行的命令后面添加 **&**,表示你将该命令放到**后台中执行**。
* ctrl + z，将当前的命令放到**后台中暂停**。（ctrl + c 是直接强制中断执行）
* jobs，查看后台的状态。
  * -l，同时列出 pid
  * 输出结果中，[1][2].. 代表任务编号。**+ 号**则表示最近那个被放到后台的任务，**- 号**表示最近第二个被放到后台的任务。其他则不显示。

![image (6)](niao-ge-linux-dishes-study-record/image (6).png)

* fg，（foreground），将后台任务取出到前台运行，不加参数默认取 + 号的那个任务。
  * fg  jobNumber，取出对应编号的任务到前台执行。
* bg，将任何在后台中任务的状态变为“**后台中执行**”。用法和 fg 类似。

**连接终端的个人 bash 的后台和整个系统的后台是两个概念**。**你在某个特定的 bash 下将任务放到后台运行，当你与主机退出连接的时候，该后台任务会中断，而不是你想的那样会一直运行。想要在整个主机中运行后台任务的话可以使用 nohup 命令。**


### 进程管理

同样的进程查看也是，当你连接主机，登录到一个 bash 下之后你执行的命令产生的子进程一般只和该 bash 下的父进程有关。

* 你可以使用 **ps -l** 查看只和自己的 bash 有关的进程。

![image (7)](niao-ge-linux-dishes-study-record/image (7).png)

    * F：process flags，进程标识。用来说明进程的权限。
    * S：STAT。
        * R，Running
        * S，Sleep。该进程处于睡眠状态（idle），但可以被唤醒（signal）
        * D：不可被唤醒的睡眠状态。可能在等待 I/O。
        * T：Stop。可能被手动暂停。
        * Z：Zombie。进程终止，但是无法被清出内存。
    * PRI/NI，优先级，越小优先级越高。

* 使用 **ps aux** 查看整个系统的进程。

* **top。top**执行后，会处于动态查看系统状态的界面，如下图。
  
  ![image (8)](niao-ge-linux-dishes-study-record/image (8).png)
  
  * 第 3 行中的 wa 指的是系统 I/O 的 wait，平时可以多注意这一项。
  * 最后一行的交换区（虚拟内存）用量也需要注意，用的越多说明系统内存可能告急。
    * -d 秒数，top 更新的频率，默认 5s。
    * -b，按照批次输出 top 结果，可以配合 -n。
    * -n 次数，执行几次 top 命令的结果。
    * -p pid，只看特定 pid 的执行结果。
  
* free，查看内存信息，-h，更可读的方式

### 



