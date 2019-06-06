---
layout: post
title: iscc2016 misc writeup
category: writeup
tags: ctf
description:
---

0x00 引子
----
jarvisoj浙江大学系统安全实验室(USS Lab.)学生Jarvis所开发的一个CTF在线答题系统.
发现适合我这样的小菜玩于是花了一些时间玩了玩.

0x01 shell流量分析 100
----

分析一下shell流量，得到flag

[+_+.rar][1]

![aaaaa.png][2]

发现2333端口是一个bash反弹的shell
cat 几个文件function.py    flag

```
#!/usr/bin/env python
# coding:utf-8
__author__ = 'Aklis'

from Crypto import Random
from Crypto.Cipher import AES

import sys
import base64


def decrypt(encrypted, passphrase):
  IV = encrypted[:16]
  aes = AES.new(passphrase, AES.MODE_CBC, IV)
  return aes.decrypt(encrypted[16:])


def encrypt(message, passphrase):
  IV = message[:16]
  length = 16
  count = len(message)
  padding = length - (count % length)
  message = message + '\0' * padding
  aes = AES.new(passphrase, AES.MODE_CBC, IV)
  return aes.encrypt(message)


IV = 'YUFHJKVWEASDGQDH'

message = IV + 'flag is hctf{xxxxxxxxxxxxxxx}'


print len(message)

example = encrypt(message, 'Qq4wdrhhyEWe4qBF')
print example
example = decrypt(example, 'Qq4wdrhhyEWe4qBF') 
print example
```

```
mbZoEMrhAO0WWeugNjqNw3U6Tt2C+rwpgpbdWRZgfQI3MAh0sZ9qjnziUKkV90XhAOkIs/OXoYVw5uQDjVvgNA==
```
然后观察

![cccc.png][3]

猜测是base64解码然后在解密 key有
然后修改脚本

![QQ截图20161231145506.png][4]

解密得到flag

-----------

0x02 Struts2漏洞 500
----

[struts2.rar][5]

既然是分析压缩包中的数据包文件并获取flag。flag为32位大写md5。
那么一定是http 协议咯
方法一：
![QQ截图20161231150832.png][6]

方法二：
strings 大法

---------
0x03 Webshell分析 300
----

[findwebshell.rar][7]

分析压缩包中的数据包文件并获取flag。flag为32位小写md5。

![1.png][8]

![2.png][9]

发现请求了这个
然后返回了一个包
```
aHR0cHM6Ly9kbi5qYXJ2aXNvai5jb20vY2hhbGxlbmdlZmlsZXMvQWJUekEyWXRlRmpHaFBXQ2Z0cmFvdVZEM0I2ODRhOUEuanBn
```
解密是一个二维码的地址

![QQ截图20161231151803.png][10]

--------
0x04 远程登录协议 100
--

分析压缩包中的数据包文件并获取flag。flag为32位小写md5。


telnet 协议

[telnet.rar][11]

法1：
![QQ截图20161231152121.png][12]

法2：
一样strings 大法能找到

---------
0x05 简单网管协议 100
--
[simple_protocol.rar][13]

打开发现snmp协议 

![QQ截图20161231152525.png][14]

strings 大法

---------
0x06 取证2 350
--
还记得取证那题吗？既然有了取证神器，这里有一个可疑文件以及该存储该文件电脑的一个内存快照，那么接下来我们实战一下吧。

太大了就不发上来了
解压发现一个快照文件和一个不知道是什么的文件

用volatility 分析发现mem.vmem是个win xp的内存快照
```
volatility -f mem.vmem  pslist
```

![21.png][15]

发现一个TrueCrypt.exe进程
于是搜索了一下

![222..png][16]

找到了神器
Elcomsoft Forensic Disk Decryptor
于是dump TrueCrypt进程

![22222.png][17]

装好Elcomsoft Forensic Disk Decryptor

![121212.png][18]

选择那个不知道是什么文件的东西试试

然后得出key
挂载文件得到flag

![4.png][19]

ps:还以为是分析恶意进程。。。
http://www.behindthefirewalls.com/2013/12/stuxnet-trojan-memory-forensics-with_16.html
膜拜国外取证大佬
------
0x07 SCAN 100
--

[capture.rar][20]

有人在内网发起了大量扫描，而且扫描次数不止一次，请你从capture日志分析一下对方第4次发起扫描时什么时候开始的，请提交你发现包编号的sha256值(小写)。


Hint1: 请提交PCTF{包编号的sha256}

打开发现后缀是log 用file 看了一下是tcpdump 得到的流量 于是改成cap

扫描对象都是99但是每次都是先ping看看是否存活
于是就筛选

![cxzcxz.png][21]

一个一个的试也不多 但是猜测一个ip算为一次的话 那么第四个ip的包编号就是我们要的.

-------

ps:我好菜啊除了web 其他的难度大一点的都不会做了.求大师傅指导..

  [1]: https://img.scanfsec.com/img/2016122045664263.rar
  [2]: https://img.scanfsec.com/img/201612886251388.png
  [3]: https://img.scanfsec.com/img/2016123717368855.png
  [4]: https://img.scanfsec.com/img/2016124195643165.png
  [5]: https://img.scanfsec.com/img/2016123447920927.rar
  [6]: https://img.scanfsec.com/img/201612229744331.png
  [7]: https://img.scanfsec.com/img/201612760413209.rar
  [8]: https://img.scanfsec.com/img/2016122471110323.png
  [9]: https://img.scanfsec.com/img/2016124097463853.png
  [10]: https://img.scanfsec.com/img/2016121504406345.png
  [11]: https://img.scanfsec.com/img/2016123004296419.rar
  [12]: https://img.scanfsec.com/img/201612393395944.png
  [13]: https://img.scanfsec.com/img/201612814945111.rar
  [14]: https://img.scanfsec.com/img/2016122090328382.png
  [15]: https://img.scanfsec.com/img/2016124225127155.png
  [16]: https://img.scanfsec.com/img/2016121339722191.png
  [17]: https://img.scanfsec.com/img/2016122365497541.png
  [18]: https://img.scanfsec.com/img/2016121261641888.png
  [19]: https://img.scanfsec.com/img/2016122995960936.png
  [20]: https://img.scanfsec.com/img/2016122821167613.rar
  [21]: https://img.scanfsec.com/img/2016124281839083.png
