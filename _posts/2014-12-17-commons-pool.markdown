---
layout: post
category: "java"
title: "关于commons-pool的代码量"
tags: [commons,pool,cloc,redis]
---

今天在公司阿里云主机上搭建了redis集群，采用的是twitter开源的twemproxy(nutcracker)。
这玩意真是折腾，得先安装autoconf、libtool。对autoconf的版本还有要求。
onfigure.ac:8: error: Autoconf version 2.64 or higher is required
而云主机上yum install autoconf安装的版本达不到要求，so，download http://ftp.gnu.org/gnu/autoconf/autoconf-2.69.tar.gz
然后yum install libtool

twemproxy处理单点故障有点坑，server_retry_timeout设置小了，kill掉一台redis，报错 ERR Connection refused

\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-上面扯淡了\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-\-

一开始git clone是被17104惊讶了
<pre class="prettyPrint">
git clone https://github.com/apache/commons-pool.git
Cloning into 'commons-pool'...
remote: Counting objects: 17104, done.
remote: Total 17104 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (17104/17104), 5.70 MiB | 53.00 KiB/s, done.
Resolving deltas: 100% (7216/7216), done.
Checking connectivity... done.
</pre>

接下来用cloc统计了下代码量

cloc commons-pool
     120 text files.
     117 unique files.
      16 files ignored.

http://cloc.sourceforge.net v 1.62  T=0.68 s (154.2 files/s, 38443.1 lines/s)
----------------------------------------------------------------------------------------
Language                              files          blank        comment           code
----------------------------------------------------------------------------------------
Java                                     78           2365           8024          12733
XML                                      16            115            334           1618
Maven                                     1             17             19            280
Ant                                       1             26             24            138
HTML                                      4              8             59            125
Velocity Template Language                1             14              0            110
Bourne Shell                              4              9            122             37
----------------------------------------------------------------------------------------
SUM:                                    105           2554           8582          15041
----------------------------------------------------------------------------------------

java代码12733行， 而单单统计main中的java代码只有4854行（测试代码7811行，囧囧o(╯□╰)o），这个代码量是可以接受的。
