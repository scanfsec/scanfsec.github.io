---
layout: post
title: pwnhub迷web_writeup
category: writeup
tags: ctf
description:
---
201612
by:scanf
首先膜拜rr菊苣
什么提示都没有就一个ip
于是就先扫了一下端口
发现开放了 80 1234 22
加上README.md里面的信息


![ctf0.png][1]

又不能扫于是猜测是不是ssh弱口令
测试 发现proxy proxy 可以登录 但是连上就断 于是试了试ssh隧道

![ctf2.png][2]

于是扫了一发本机

![ctf3.png][3]

发现开了9000端口 猜测是FastCGI远程命令执行
于是要找一个.php
找了一个ubuntu搜索了一下发现pear的安装目录在/usr/share/php/

![ctf4.png][4]

于是就得到了flag

![ctf5.png][5]

  [1]: https://img.scanfsec.com/img/2016123948984521.png
  [2]: https://img.scanfsec.com/img/20161219462082.png
  [3]: https://img.scanfsec.com/img/2016122532484078.png
  [4]: https://img.scanfsec.com/img/2016122343710200.png
  [5]: https://img.scanfsec.com/img/201612440555346.png
