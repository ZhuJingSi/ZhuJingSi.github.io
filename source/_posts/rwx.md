---
title: 关于rwx（读、写、执行）的理解
date: 2016-03-01 00:46:42
categories: 
  - linux
tags: 
  - rwx
  - linux
---

#### 问题

今天自己手工起了一个 wordpress，发现在上传图片的时候报了一个错：Unable to create directory wp-content/uploads/2016/03. It its parent directory writable by the server? 原因在报错中已经比较清楚了，就是上级目录没有被写的权限，因此不能创建 uploads 目录存放图片。

<!--more-->

#### 查找资料后有人给出了解决方案：

1. 将 wordpress 目录的权限改为 777 （ 所有人可读可写可执行 ）；
   
   ``` shell
   chmod -R 777 wordpress
   ```
   
2. 上传一次图片，uploads 目录被创建，因此可以获知 uploads 目录的属主（ ls -l，假设是 a ）；
   
3. 将整个 wordpress 目录的属主和组（ EUID 和 EGID ）改为 a;
   
   ``` shell
   chown a:a wordpress
   ```
   
4. 再将 wordpress 目录的权限改为 755 （ 对属主：可读可写可执行，对组和其它：可读可执行）；
   
   ``` shell
   chmod -R 755 wordpress
   ```

**这种做法当然是可行的，但是在原理上绕了一个不必要的弯。首先，要弄懂是谁去创建这个 uploads 文件的，这并不需要先让它创建了我们再看是谁。**

#### 探索正确的解决方案

我们都知道，客户端请求是发给服务器的，服务器又是通过 Web服务器软件来负责接受并转发给某个服务器端程序去分析和处理的。在这里，是通过 Apache 将请求转发给了 php 处理，那么我们先认为是 php 的进程创建了这个 uploads，这个 php 进程的属主也就自然而然是 uploads 的属主了。

**现在试着查看 php 进程：**

``` shell
ps aux|grep php
```

**居然没有找到一个 php 的进程！**那我们再看看 apache：

``` shell
ps aux|grep apache
```

这次拿到了一大堆：

``` shell
root      7834  0.0  1.5 180868 13840 ?   Ss   Mar05   0:00 /usr/sbin/apache2 -k start
www-data 21707  0.0  5.3 218080 48152 ?  S    Mar05   0:06 /usr/sbin/apache2 -k start
www-data 21708  0.0  3.5 201880 32244 ?  S    Mar05   0:02 /usr/sbin/apache2 -k start
www-data 21709  0.0  3.4 200280 30504 ?  S    Mar05   0:03 /usr/sbin/apache2 -k start
www-data 21722  0.0  3.6 202872 33140 ?  S    Mar05   0:02 /usr/sbin/apache2 -k start
```

这就说明了，是 Apache 在启动后将请求派发给了它的各个进程处理，这些进程的属主叫 “**www-data**”，就是它了！

接下来进行最后一步：将 wordpress 目录的属主和组（ UID 和 GID ）改为 “www-data”。

``` shell
chown www-data:www-data wordpress
```

**总结下来只需要两步就可到位，也不用将权限改来改去。**

#### 反思

**这里有个思路，当 a 进程想写 b 文件 而却没有权限的时候要怎么做才能成功？**

 有三个办法：

1. 改 b 的权限，使 b 可以让所有人可读可写； 
2. 改 a 的属主，将 a 的属主改成和 b 一样；
3. 改 b 的属主，将 b 的属主改成歌 a 一样；

那么哪一种比较好呢？答案是**第三种**。

**第一种**，让 b 可以被所有人读写会带来潜在的安全隐患，指不定就被哪个恶意程序篡改导致不可使用；

**第二种**，若 b 文件的权限是 root，a 也要改成 root，这时候 a 就可以去篡改系统中任意一个文件，若 a 被黑客劫持，后果不堪设想，这种更加危险；

**第三种**，b 的属主改成 a 的，虽然这可能导致其它应用的进程没法写它，但一般这个文件只要不是公共的而是属于某一个应用独享的，那么也就只有这个应用会用到它，且不用担心安全问题；

#### 总结知识点：

r、w、x 这三种权限，若为文件的属性就是字面意思。若是一个目录的属性则：

**r**：该目录下的文件或目录是否可列出来看到；

**w**：该目录下是否可以添加或删除文件或目录；

**x**：该目录下的文件或目录是否可以被 access；

example:

目录 a 的权限为 drwxr-x--x（ 751，属主：可读可写可执行、组：可读可执行、其它：可执行）属主不必说，什么操作都可以；同组的可以通过 ls a  命令查看目录，也可以进入或切换到这个目录；其它人只可以切换到这个目录但看不到目录下的东西，也不能在其中创建或删除东西；