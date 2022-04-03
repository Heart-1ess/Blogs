---
layout: post
title: CobaltStrike学习笔记——Veil制作免杀木马
date: 2022-04-03 14:16:00 +0800
last_modified_at: 2022-04-03 14:16:00 +0800
tags: [Web]
toc:  true
---
## 0x01 下载安装Veil
采用docker安装，先在kali中安装docker

添加清华镜像源

```
curl -fsSL https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian/gpg | sudo apt-key add -
```

配置docker apt：

```
echo 'deb https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian/ buster stable' | sudo tee /etc/apt/sources.list.d/docker.list
```

更新apt：

```
sudo apt-get update
```

安装docker，先检测是否安装过docker，如果安装过需要先卸载旧版本：

```
sudo apt-get docker docker-engine docker.io
```

若提示

```
E: 无效的操作docker
```

则可以开始安装docker，否则需要先卸载旧版本

进行安装：

```
sudo apt-get install docker-ce
```

安装后添加docker加速镜像地址`vim /etc/docker/daemon.json`

```
{
    "registry-mirrors": [
        "https://1nj0zren.mirror.aliyuncs.com",
        "https://docker.mirrors.ustc.edu.cn",
        "http://f1361db2.m.daocloud.io",
        "https://registry.docker-cn.com"
    ]
}
```

然后重启docker服务

```
systemctl daemon-reload
systemctl restart docker
```

拉取veil镜像

```
docker pull mattiasohlsson/veil
```

拉取成功后，启动容器，并将生成免杀文件的目录映射到宿主机的/tmp目录中

```
docker run -it -v /tmp/veil-output:/var/lib/veil/output:Z mattiasohlsson/veil
```

![Cobalt-1](https://raw.githubusercontent.com/Heart-1ess/Heart_1ess-s-CTF-Note/master/blogs/assets/Cobalt-1.png)

如图所示，启动成功

## 0x02 CobaltStrike生成载荷

启动CobaltStrike并在客户端上线，设置Listener为监听机地址，并进行载荷生成

![Cobalt-3](https://raw.githubusercontent.com/Heart-1ess/Heart_1ess-s-CTF-Note/master/blogs/assets/Cobalt-3.png)

![Cobalt-4](https://raw.githubusercontent.com/Heart-1ess/Heart_1ess-s-CTF-Note/master/blogs/assets/Cobalt-4.png)

生成界面选择Veil，监听器选择刚刚设置的监听器，生成载荷

![Cobalt-5](https://raw.githubusercontent.com/Heart-1ess/Heart_1ess-s-CTF-Note/master/blogs/assets/Cobalt-5.png)


## 0x03 生成免杀马

打入指令启动免杀模式

```
use 1
```

而后通过`list`指令查看免杀方式，选择aes

![Cobalt-2](https://raw.githubusercontent.com/Heart-1ess/Heart_1ess-s-CTF-Note/master/blogs/assets/Cobalt-2.png)

```
use 29
```

而后选择

```
generate
```

后面选择自定义shellcode，即3号选项并打入CobaltStrike生成的payload

![Cobalt-6](https://raw.githubusercontent.com/Heart-1ess/Heart_1ess-s-CTF-Note/master/blogs/assets/Cobalt-6.png)

并且采用PyInstaller的模式创建一个exe文件

自此木马创建完成

放入winxp靶机中运行，成功上线

