---
layout: post_layout
title: Rsync 未授权访问漏洞复现
time: 2019年05月28日 星期二
location: 杭州
pulished: true
excerpt_separator: "### 1 漏洞原理"
---
### 0 背景

---

未授权访问漏洞，指需要授权的地址、页面缺乏认证，导致任意用户可以直接访问，引发的问题轻则敏感信息泄露，重则命令执行。

Rsync，linux 下一款远程同步软件，能同时同步多台计算机的文件及目录，并能保持源文件的**权限**、时间、软硬连接等附加信息。

---

### 1 漏洞原理

---

#### 1. 可读

Rsync 未配置认证用户名密码，导致任意用户可以查看目录、下载文件。

客户端命令：

>rsync [-avz] [--port=xxx] IP::[目录/文件] [目录]

-a --archive 归档模式，表示递归传输并保持文件属性。

-v 显示 rsync 过程中详细信息。

-z 传输时进行压缩提高效率。

最常用的选项组合是 "-avz"，即压缩和显示部分信息，并以归档模式传输。

--port 连接 daemon 时使用的端口号，**默认为873端口**。

IP 为服务器 ip，:: 后跟目录则显示目录列表，跟目录+文件则下载该文件。

最后的目录为下载到的本地目录。

示例：

>rsync -avz --port=12345 192.168.1.111::test/index.php /home/user

该命令通过12345端口将 192.168.1.111 中 test/index.php 文件下载到了本地的 /home/user 中。

#### 2. 可写

若 rsync 配置文件中关闭了只读选项，即 "read only = no",则可以对服务器进行文件上传。可以上传带有777权限的 webshell 或者可以命令执行的 php 脚本。

客户端命令：

>rsync [-avz] [--port=xxx] [目录+文件] IP::[目录]

参数同上，第一处目录+文件为本地待上传的文件，第二处目录为上传目录。

---

### 2 漏洞复现

---

#### 1. 环境准备

* 一台 CentOS 7 64 位的虚拟机。

* Xmapp 安装包。

#### 2. 复现步骤

>1.将 CentOS 克隆两份，一份为 server，一份为 client。
>2.在 CentOS server 上安装 rsync，安装 xampp，将rsync的默认目录设置为 xampp 的网站根目录。
>3.在 CentOS client 上安装 rsync，下载网站的 index.php；写一个测试用的 test.php，上传至网站根目录并访问。

具体如下：

1.CentOS 连接网络

刚安装的 CentOS 不能连接网络，需要更改网卡配置。（CentOS 可以连网则跳过该步骤）

>su
>vi /etc/sysconfig/network-scripts/ifcfg-ens33

按i → 将 ONBOOT=no 改为 ONBOOT=yes → 按Esc → :wq

在物理机中打开 cmd，运行

>net start "Vmware DHCP Service"
>net start "Vmware NAT Service"

再进入 CentOS，输入

>service network restart
>ping www.baidu.com

正常访问网络。

2.安装 rsync

>yum install rsync -y

3.配置 rsync

>vi /etc/rsyncd.conf

配置基本的信息：

```
uid = root
gid = root
use chroot = yes
max connections = 4
exclude = lost+found/
transfer logging = yes
timeout = 900
ignore nonreadable = yes
dont compress = .gz .tgz .zip .z .Z .rpm .deb .bz2
[test] # 可自定义的标签名
	path = /home/ # 客户端同步的服务器默认目录
	comment = this is a test
ignore errors
read only = no # 设置是否只读
```

此处为了测试，不再配置认证用户名及密码。正确的操作是在这时添加用户名及密码。

4.安装 xmapp

访问 xampp [官网](https://www.apachefriends.org/zh_cn/index.html) ,下载 for Linux 的最新版本 xampp 的 .run 文件。下载完成后上传至服务器的 /opt/ 目录下。

或者在服务的 /opt/ 目录下直接输入命令

>wget https://www.apachefriends.org/xampp-files/7.3.5/xampp-linux-x64-7.3.5-1-installer.run

给该文件提权

>chmod -R 755 xampp-linux-x64-7.3.5-1-installer.run

安装 xmapp

>./xampp-linux-x64-7.3.5-1-installer.run

安装完成后启动 xampp

>/opt/lampp/lampp start

此时，网站的根目录为 /opt/lampp/htdocs/

5.更改 rsync 默认目录

>vi /etc/rsyncd.conf

将 path = /home/ 改为 path = /opt/lampp/htdocs/

6.启动 rsync

>systemctl start rsyncd

7.在 CentOS client 上，连接服务器的 rsync

>rsync 192.168.xxx.xxx::

发现不能访问，需要关闭服务器的防火墙

CentOS 默认的防火墙为 firewall

>systemctl stop firewalld.service

客户端即可访问

![访问成功](https://upload-images.jianshu.io/upload_images/18110176-dab7caee4351e14d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

客户端访问 test 目录

>rsync 192.168.xxx.xxx::test

![访问目录](https://upload-images.jianshu.io/upload_images/18110176-2291fc185d6844d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

客户端下载 index.php 文件到本地的 /home/ 目录中

>rsync 192.168.xxx.xxx::test/index.php /home/

![下载文件](https://upload-images.jianshu.io/upload_images/18110176-2ac135b03ad2b958.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在客户端中写一个包含了 phpinfo() 的脚本

```
<?php
    phpinfo();
?>
```

![php 脚本](https://upload-images.jianshu.io/upload_images/18110176-9180b24cff875858.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将脚本上传至服务器

>rsync a.php 192.168.xxx.xxx::test

发现上传权限不足，检查 /opt 目录的权限也没有问题，最后发现需要在服务器中关闭 enforce 模式

```
[root@localhost]# getenforce
Enforcing

[root@localhost]# setenforce 0
关闭 enforce 模式

[root@localhost]# getenforce
Permissive
```

重新上传，上传文件成功，浏览器访问 192.168.xxx.xxx/a.php

![脚本上传成功](https://upload-images.jianshu.io/upload_images/18110176-c5e87b7b4bc408a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
