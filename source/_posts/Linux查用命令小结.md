---
title: Linux常用命令小结
date: 2016-10-8 22:16:35
tags:
	- Linux
	- shell
categories: Linux
---

## 文件操作命令

### rm命令

### mkdir命令

### touch命令

### cp命令

### mv命令

### cd命令

### tar命令

### ls命令

## 文本操作命令

### cat命令

cat命令主要用来查看文件内容，创建文件，文件合并，追加文件内容等功能。

查看文件内容主要用法：
* cat f1.txt，查看f1.txt文件的内容。
* cat -n f1.txt，查看f1.txt文件的内容，并且由1开始对所有输出行进行编号。
* cat -b f1.txt，查看f1.txt文件的内容，用法与-n相似，只不过对于空白行不编号。
* cat -s f1.txt，当遇到有连续两行或两行以上的空白行，就代换为一行的空白行。
* cat -e f1.txt，在输出内容的每一行后面加一个$符号。
* cat f1.txt f2.txt，同时显示f1.txt和f2.txt文件内容，注意文件名之间以空格分隔，而不是逗号。
* cat -n f1.txt>f2.txt，对f1.txt文件中每一行加上行号后然后写入到f2.txt中，会覆盖原来的内容，文件不存在则创建它。
* cat -n f1.txt>>f2.txt，对f1.txt文件中每一行加上行号后然后追加到f2.txt中去，不会覆盖原来的内容，文件不存在则创建它。

### sort命令

sort主要用来对File参数指定的文件中的行排序，并将结果写到标准输出。如果File参数指定多个文件，那么sort命令
将这些参数连接起来，当做一个文件进行排序。

```
[root@www ~]# sort [-fbMnrtuk] [file or stdin]
选项与参数：
 -f  ：忽略大小写的差异，例如 A 与 a 视为编码相同；
 -b  ：忽略最前面的空格符部分；
 -M  ：以月份的名字来排序，例如 JAN, DEC 等等的排序方法；
 -n  ：使用『纯数字』进行排序(默认是以文字型态来排序的)；
 -r  ：反向排序；
 -u  ：就是 uniq ，相同的数据中，仅出现一行代表；
 -t  ：分隔符，默认是用 [tab] 键来分隔；
 -k  ：以那个区间 (field) 来进行排序的意思
```

**对/etc/passwd 的账号进行排序**

root@ubuntu-03:~# cat /etc/passwd | sort

sort 是默认以第一个数据来排序，而且默认是以字符串形式来排序,所以由字母 a 开始升序排序。

**/etc/passwd 内容是以 : 来分隔的，我想以第三栏来排序，该如何**

root@ubuntu-03:~# cat /etc/passwd | sort -t ":" -k 3

**默认是以字符串来排序的，如果想要使用数字排序：**

root@ubuntu-03:~# cat /etc/passwd | sort -t ":" -k 3n

**默认是升序排序，如果要倒序排序，如下**

root@ubuntu-03:~# cat /etc/passwd | sort -t ':' -k 3nr

### uniq命令

uniq命令可以去除排序过的文件中的重复行，因此uniq经常和sort合用。也就是说，为了使uniq起作用，所有的重复行必须是相邻的。
```
[root@www ~]# uniq [-icu]
选项与参数：
-i  ：忽略大小写字符的不同；
-c  ：进行计数
-u  ：只显示唯一的行
```
假设testfile内容如下：

```
cat testfile
hello
world
friend
hello
world
hello
```
直接删除未经排序的文件，将会发现没有任何行被删除
```
#uniq testfile
hello
world
friend
hello
world
hello
```
**排序文件，默认去重**
```
#cat words | sort |uniq
friend
hello
world
```
**排序之后删除了重复行，同时在行首位置输出该行重复的次数**
```
#sort testfile | uniq -c
1 friend
3 hello
2 world
```
### cut命令
cut命令可以从一个文本文件或者文本流中提取文本列。

**cut语法：**
```
[root@www ~]# cut -d'分隔字符' -f fields <==用于有特定分隔字符
[root@www ~]# cut -c 字符区间            <==用于排列整齐的信息
选项与参数：
-d  ：后面接分隔字符。与 -f 一起使用；
-f  ：依据 -d 的分隔字符将一段信息分割成为数段，用 -f 取出第几段的意思；
-c  ：以字符 (characters) 的单位取出固定字符区间；
```
**PATH变量如下**
```
root@ubuntu-03:~# echo $PATH
/usr/java/jdk/bin:/usr/java/jdk/jre/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```
**将 PATH 变量取出，我要找出第五个路径。**
```
root@ubuntu-03:~# echo $PATH | cut -d ':' -f 5
/usr/sbin
```
**将 PATH 变量取出，我要找出第三到最后一个路径。**
```
root@ubuntu-03:~# echo $PATH | cut -d ':' -f 3-
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin

```
**将 PATH 变量取出，我要找出第一到第三个路径。**
```
root@ubuntu-03:~# echo $PATH | cut -d ':' -f 1-3
/usr/java/jdk/bin:/usr/java/jdk/jre/bin:/usr/local/sbin
```

### wc命令
统计文件里面有多少单词，多少行，多少字符。wc语法：
```
[root@www ~]# wc [-lwm]
选项与参数：
-l  ：仅列出行；
-w  ：仅列出多少字(英文单字)；
-m  ：多少字符；
```
**默认使用wc统计/etc/passwd**
```
root@ubuntu-03:~# wc /etc/passwd
  30   42 1561 /etc/passwd
```
40是行数，45是单词数，1719是字节数

**wc的命令比较简单使用，每个参数使用如下：**
```
root@ubuntu-03:~# wc -l /etc/passwd
30 /etc/passwd                      #统计行数
root@ubuntu-03:~# wc -w /etc/passwd
42 /etc/passwd                      #统计单词数
root@ubuntu-03:~# wc -m /etc/passwd
1561 /etc/passwd                    #统计文件字节数
```

### grep搜索命令
Linux系统中grep命令是一种强大的文本搜索工具，它能使用正则表达式搜索文本，并把匹 配的行打印出来。grep全称是Global Regular Expression Print，表示全局正则表达式版本，它的使用权限是所有用户。

### awk命令
awk是一个强大的文本分析工具，相对于grep的查找，sed的编辑，awk在其对数据分析并生成报告时，显得尤为强大。简单来说awk就是把文件逐行的读入，以空格为默认分隔符将每行切片，切开的部分再进行各种分析处理。

awk有3个不同版本: awk、nawk和gawk，未作特别说明，一般指gawk，gawk 是 AWK 的 GNU 版本。

awk其名称得自于它的创始人 Alfred Aho 、Peter Weinberger 和 Brian Kernighan 姓氏的首个字母。实际上 AWK 的确拥有自己的语言： AWK 程序设计语言 ， 三位创建者已将它正式定义为“样式扫描和处理语言”。它允许您创建简短的程序，这些程序读取输入文件、为数据排序、处理数据、对输入执行计算以及生成报表，还有无数其他的功能。

**使用方法**
``` shell
awk '{pattern + action}' {filenames}
```
尽管操作可能会很复杂，但语法总是这样，其中 pattern 表示 AWK 在数据中查找的内容，而 action 是在找到匹配内容时所执行的一系列命令。花括号（{}）不需要在程序中始终出现，但它们用于根据特定的模式对一系列指令进行分组。 pattern就是要表示的正则表达式，用斜杠括起来。

awk语言的最基本功能是在文件或者字符串中基于指定规则浏览和抽取信息，awk抽取信息后，才能进行其他文本操作。完整的awk脚本通常用来格式化文本文件中的信息。

通常，awk是以文件的一行为处理单位的。awk每接收文件的一行，然后执行相应的命令，来处理文本。

**调用awk**
```
1.命令行方式
awk [-F  field-separator]  'commands'  input-file(s)
其中，commands 是真正awk命令，[-F域分隔符]是可选的。 input-file(s) 是待处理的文件。
在awk中，文件的每一行中，由域分隔符分开的每一项称为一个域。通常，在不指名-F域分隔符的情况下，默认的域分隔符是空格。

2.shell脚本方式
将所有的awk命令插入一个文件，并使awk程序可执行，然后awk命令解释器作为脚本的首行，一遍通过键入脚本名称来调用。
相当于shell脚本首行的：#!/bin/sh
可以换成：#!/bin/awk

3.将所有的awk命令插入一个单独文件，然后调用：
awk -f awk-script-file input-file(s)
其中，-f选项加载awk-script-file中的awk脚本，input-file(s)跟上面的是一样的
```

###入门实例
**显示最近登录的5个帐号**
```
root@ubuntu-03:~# last -n 5 | awk '{print $1}'
root
root
root

wtmp
```
awk工作流程是这样的：读入有'\n'换行符分割的一条记录，然后将记录按指定的域分隔符划分域，填充域，$0则表示所有域,$1表示第一个域,$n表示第n个域。默认域分隔符是"空白键" 或 "[tab]键",所以$1表示登录用户，$3表示登录用户ip,以此类推

**如果只是显示/etc/passwd的账户和账户对应的shell,而账户与shell之间以tab键分割**
```
root@ubuntu-03:~# cat /etc/passwd | awk -F ':' '{print $1"\t"$7}'
root    /bin/bash
daemon  /usr/sbin/nologin
bin     /usr/sbin/nologin
sys     /usr/sbin/nologin
sync    /bin/sync
```
**如果只是显示/etc/passwd的账户和账户对应的shell,而账户与shell之间以逗号分割,而且在所有行添加列名name,shell,在最后一行添加"blue,/bin/nosh"。**
```
root@ubuntu-03:~# cat /etc/passwd | awk -F ':' 'BEGIN{print "name, shell"}{print $1"\t"$7}END{print "blue,/bin/bash"}'
name, shell
root    /bin/bash
daemon  /usr/sbin/nologin
bin     /usr/sbin/nologin
sys     /usr/sbin/nologin
sync    /bin/sync
blue, /bin/bash
```
awk工作流程是这样的：先执行BEGING，然后读取文件，读入有/n换行符分割的一条记录，然后将记录按指定的域分隔符划分域，填充域，$0则表示所有域,$1表示第一个域,$n表示第n个域,随后开始执行模式所对应的动作action。接着开始读入第二条记录······直到所有的记录都读完，最后执行END操作。

**搜索/etc/passwd有root关键字的所有行，并显示对应的shell**
```
root@ubuntu-03:~# awk -F: '/root/{print $7}' /etc/passwd
/bin/bash
```
这里指定了action{print $7}

**AWK内置变量**
```
ARGC               命令行参数个数
ARGV               命令行参数排列
ENVIRON            支持队列中系统环境变量的使用
FILENAME           awk浏览的文件名
FNR                浏览文件的记录数
FS                 设置输入域分隔符，等价于命令行 -F选项
NF                 浏览记录的域的个数
NR                 已读的记录数
OFS                输出域分隔符
ORS                输出记录分隔符
RS                 控制记录分隔符
```
**统计/etc/passwd:文件名，每行的行号，每行的列数，对应的完整行内容:**
```
root@ubuntu-03:~# awk -F ':' '{print "filename:" FILENAME ",linenumber:" NR ",columns:" NF "linecontent:" $0}' /etc/passwd
filename:/etc/passwd,linenumber:1,columns:7linecontent:root:x:0:0:root:/root:/bin/bash
filename:/etc/passwd,linenumber:2,columns:7linecontent:daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
filename:/etc/passwd,linenumber:3,columns:7linecontent:bin:x:2:2:bin:/bin:/usr/sbin/nologin
filename:/etc/passwd,linenumber:4,columns:7linecontent:sys:x:3:3:sys:/dev:/usr/sbin/nologin

```
**使用printf替代print,可以让代码更加简洁，易读**
```
awk  -F ':'  '{printf("filename:%10s,linenumber:%s,columns:%s,linecontent:%s\n",FILENAME,NR,NF,$0)}' /etc/passwd
```

### awk编程
**下面统计/etc/passwd的账户人数**
```
root@ubuntu-03:~# awk '{count++} END{print "total is:" count}' /etc/passwd
total is:30
```

count是自定义变量。之前的action{}里都是只有一个print,其实print只是一个语句，而action{}可以有多个语句，以;号隔开。

这里没有初始化count，虽然默认是0，但是妥当的做法还是初始化为0:
```
awk 'BEGIN{count=0; print "count is:" count} {count++} END{print "total is:" count}' /etc/passwd
```

**awk的循环语句**
awk中的循环语句同样借鉴于C语言，支持while、do/while、for、break、continue，这些关键字的语义和C语言中的语义完全相同。

**awk的数组**
因为awk中数组的下标可以是数字和字母，数组的下标通常被称为关键字(key)。值和关键字都存储在内部的一张针对key/value应用hash的表格里。由于hash不是顺序存储，因此在显示数组内容时会发现，它们并不是按照你预料的顺序显示出来的。数组和变量一样，都是在使用时自动创建的，awk也同样会自动判断其存储的是数字还是字符串。一般而言，awk中的数组用来从记录中收集信息，可以用于计算总和、统计单词以及跟踪模板被匹配的次数等等。

显示/etc/passwd的账户
```
awk -F ':' 'BEGIN {count=0;} {name[count] = $1;count++;}; END{for (i = 0; i < NR; i++) print i, name[i]}' /etc/passwd
0 root
1 daemon
2 bin
3 sys
4 sync
5 games
```


## 权限操作命令
Linux一般将文件可存取访问的身份分为3个类别：owner, group, others且三种身份各有read,write,execute等权限。

### 用户和用户组
**文件所有者**
可以设置除本人之外的用户无法查看文件的内容
**用户组**
假设A组有a1,a2,a3三个成员，B组有b1,b2成员，共同完成一份报告F。设置了适当的权限之后，A,B可以相互修改数据。C团体无法查看也无法修改。
**其他人**
（相对概念），比如f是A组以外的成员，那么f相对于A组成员就是其他人。
**超级用户root**
他拥有最大的权限，也管理普通用户
**相关文件**
在linux系统中，默认的系统账户和普通账户信息记录在/etc/passwd文件中

个人密码在/etc/shadow下

用户组名称记录在/etc/group下，所以这三个文件不能随便删除

### linux文件权限的概念
ls -la可查看文件详情
```
drwxr-xr-x 4 root root 4096 Feb 21 19:18 neo4j
  |  |  |  |   |    |    |        |
文件权限  连接数 |    |  文件大小  文件最后被修改时间
           文件所有者 |
                 文件所属用户组
```
d rwx r-x r-x 中d代表文件类型，rwx代表文件所有者权限，r-x代表文件所属用户组权限，---代表其他人权限

* r:read 可读
* w:write 可写
* x:execute 可执行
* -：没有对应权限

### 改变文件属性的权限
* chgrp：改变文件所属用户组
* chown: 改变文件所有者
* chmod: 改变文件权限

**改变所属用户组**
chgrp(change group) 不过，要被改变的组名必须在/etc/group文件内才行，否则会出错。
```
chgrp [-R] 文件名/目录名 -R：进行递归，可以修改目录下的文件
```
**改变文件所有者**
chown(change owner) 用户名必须存在于/etc/passwd中才行
```
chown [-R] 账号名称 文件名/目录名
chown [-R] 账号名称：用户组名 文件名/目录名
```
**改变权限**
chmod(change mode)有两种改变权限的方式：

1.数字类型改变文件权限

linux的基本权限有9个，即owner,group,others三中身份都有自己的r/w/x权限，各种权限对应的数字是：r-4,w-2,x-1。

例如：-rw-r--r--

owner=rw-=4+2+0=6, group=r--=4+0+0=4,others=r--=4+0+0=4
```
chmod [-R] xyz 文件名/目录名
chmod 777 test.txt
```

2.符号类型改变权限
u,g,o代表三中身份，a代表all，即全部身份

修改test.txt的权限为rwxr-xr-x:
```
chmod u=rwx,g=r-x,o=r-x test.txt
```
去掉test.txt所有身份的x权限
```
chmod a-x test-txt
```
添加test.txt所有身份的x权限
```
chmod a+x test.txt
```
## 状态查看命令

## 远程操作命令

### ssh命令

### scp命令