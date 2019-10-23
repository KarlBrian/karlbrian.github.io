---
layout: post_layout
title: 利用 DoH 协议的恶意软件 Godlua
time: 2019年10月23日 星期三
location: 杭州
pulished: true
excerpt_separator: "比如你在地址栏输入"
---

### **0 背景**

#### **1.DoH**

DoH，全称 DNS over HTTPS。

我们知道 HTTPS 协议可以有效地阻止中间人攻击，也可以防止中间人或者运营商监测用户实时的访问信息。目前很多运营商会通过流量劫持的方式在用户访问的页面里插入广告，使用 HTTPS 加密的网页则不会受到影响。

而在 DNS 领域此前都是没有加密的，即便网页是 HTTPS 连接，运营商依然可以看到用户浏览网站的网页地址。DoH 是专门为 DNS 服务器推出的 TLS 加密功能，从用户发出访问请求开始全程加密，阻止运营商查看网页地址。比如你在地址栏输入 microsoft.com，浏览器不再进行传统 DNS 查询，而是通过 HTTPS 向某个服务器查询。这样，整个访问过程加密的，对第三方不可见。

目前 Chrome 和火狐均已支持 DoH，让用户在浏览网页时可以更好的保护自己隐私。

#### **2.Godlua**

2019年4月24号，有研究人员发现了一个可疑的 ELF 文件，该文件被一部分杀软误识别为挖矿程序。通过详细分析，确定为一款基于 Lua 编程语言的恶意后门软件。因为这个样本加载的 Lua 字节码文件幻数为 “God”，所以将它命名为 Godlua Backdoor。

Godlua Backdoor 会使用硬编码域名、Pastebin.com、GitHub.com 和 DNS TXT 记录等方式，构建存储 C2 地址的冗余机制。同时，它使用 HTTPS 加密下载 Lua 字节码文件，使用 DNS over HTTPS 获取 C2 域名解析，保障 Bot 与 Web Server 和 C2 之间的安全通信。

研究人员监测到 Godlua Backdoor 存在2个版本，并且在持续更新。研究人员还观察到攻击者会通过 Lua 指令，动态运行 Lua 代码，并对一些网站发起 HTTP Flood 攻击。

![两个版本对比图](http://r.photo.store.qq.com/psb?/V12ix5dK0c8VFD/gePX3wLkufNcbbMNausBH2BPnCIPg7pQo2chrA.MSwo!/o/dMMAAAAAAAAA&ek=1&kp=1&pt=0&bo=vAJcALwCXAADEDU!&tl=1&su=0258079041&tm=1571817600&sce=0-12-12&rf=2-9)

#### **1 理解**

#### **1.什么是 C2 服务器**

C&C 服务器（即 C2 服务器）的全称是 Command and Control Server，翻译过来就是命令和控制服务器。C&C 服务器使目标机器可以接收来自服务器的命令，从而达到服务器控制目标机器的目的。该方法常用于病毒木马控制被感染的机器。**简单地说，C&C 服务器在僵尸网络中好比一个人的大脑，而被感染 bot 的僵尸计算机就好比人的手脚，大脑统一发送指令，手脚接受并执行指令。**

#### **2.什么是硬编码**

就是将一些数据（数值，字符串）嵌在代码逻辑当中。比如：

```
var msg = a + “你好”；// 其中，“你好”就是硬编码。
```

硬编码数据通常只能通过编辑源代码和重新编译可执行文件来修改。**也就是说，Godlua 将一些 C2 域名写死在自己的程序当中。**

#### **3.什么是DNS TXT记录**

* 示例：ns1.exmaple.com. IN TXT "联系电话：XXXX"

* 解释：【domain】 IN TXT 【任意字符串】

一般指某个主机名或域名的说明，或者联系方式，或者标注提醒等等。

#### **4.什么是冗余机制**

所谓冗余机制，就是指备份。当主要的 C2 服务器地址链连接出现问题时，冗余的方式可以立刻使用来替代主要连接方式。

#### **5.总结**

Godlua 的获利方式目前被判定为使被感染机器作为发动 ddos 攻击的机器人。该病毒通过一些手段（硬编码域名、Pastebin.com、GitHub.com 和 DNS TXT 记录等），保证了被感染的机器与发布指令的“大脑”—— C2 服务器的连通性，此外恶意利用 DoH 技术，保证了 bot 安全地获取 C2 服务器的域名解析。

### **2 分析**

#### **1.Version 201811051556**

这是被发现的 Godlua Backdoor 的早期实现版本(201811051556)，它主要针对 Linux 平台，并支持2种 C2 指令，分别是执行 Linux 系统命令和执行自定义文件。它通过硬编码域名和 Github 项目描述2种方式来存储 C2 地址。

![C2冗余机制](http://r.photo.store.qq.com/psb?/V12ix5dK0c8VFD/recQMswSLmUFj1vtNBDy836BCEKxHzN**CWng5M9gkY!/o/dL8AAAAAAAAA&ek=1&kp=1&pt=0&bo=vAJSAbwCUgEDEDU!&tl=1&su=0233680257&tm=1571817600&sce=0-12-12&rf=2-9)

#### **2.Version 20190415103713 ~ 20190621174731**

它是 Godlua Backdoor 当前活跃版本，主要针对 Windows 和 Linux 平台，通过 Lua 实现主控逻辑并主要支持5种 C2 指令。

![C2冗余机制](http://r.photo.store.qq.com/psb?/V12ix5dK0c8VFD/RwR9Y7T7IE*T0z.qwam8gWbwYlzDNgAFnv4hOoNLC1s!/o/dLYAAAAAAAAA&ek=1&kp=1&pt=0&bo=vAIBArwCAQIDEDU!&tl=1&su=0144074369&tm=1571817600&sce=0-12-12&rf=2-9)

第一阶段：第一阶段的 URL 存储有3种冗余机制，分别是将该信息通过硬编码密文、Github 项目描述和 Pastebin 文本存储。在解密得到第一阶段的 URL 后会下载 start.png 文件，它实际上是 Lua 字节码。Bot 会把它加载到内存中并运行然后获取第二阶段的 URL。

第二阶段：第二阶段的 URL 存储有2种冗余机制，分别是将该信息通过 Github 项目文件和 DNS TXT 存储。在解密得到第二阶段的 URL 后会下载 run.png 文件，它也是 Lua 字节码。Bot 会把它加载到内存中并运行然后获取第三阶段的 C2 服务器的域名。

第三阶段：C2 服务器的域名硬编码在 Lua 字节码文件（run.png）中，bot 通过 DNS Over HTTPS 请求获取 C2 域名的 A 记录。

#### **3.Lua 脚本分析**

Godlua Backdoor Bot 样本在运行中会下载许多 Lua 脚本，可以分为运行、辅助、攻击三大类：

* 运行：start.png、run.png、quit.png、watch.png、upgrade.png、proxy.png
* 辅助：packet.png、curl.png、util.png、utils.png
* 攻击：VM.png、CC.png

#### **4.Lua 幻数**

解密后的文件以 upgrade.png 为例，是 pre-compiled code，高亮部分为文件头。

![文件头](http://r.photo.store.qq.com/psb?/V12ix5dK0c8VFD/EsUQRxA10yhq.HQFKqMEldeJxRRlJkzABLuQ*uo6JSI!/o/dMUAAAAAAAAA&ek=1&kp=1&pt=0&bo=vAJbALwCWwADEDU!&tl=1&su=0163014561&tm=1571817600&sce=0-12-12&rf=2-9)

可以发现幻数从 Lua 变成了 God，虽然样本中有 "\$LuaVersion: God 5.1.4 C$$LuaAuthors: R. \$" 字符串，但事实上所采用的版本并不是 5.1.4，具体版本无法确定，但可以肯定的是大于 5.2。

### **3 处置建议**

目前还没有完全看清 Godlua Backdoor 的传播途径，但研究者发现一些 Linux 用户是通过 confluence 漏洞利用（CVE-2019-3396）感染的，建议排查并修复该漏洞。此外建议对 Godluad Backdoor 相关 IP，URL 和域名进行监控和封锁。

#### **1.Confluence 远程代码执行漏洞（CVE-2019-3396）**

官方已修复该漏洞，请到官网下载无漏洞版本：[https://www.atlassian.com/](https://www.atlassian.com/)

#### **2.IoC list**

样本MD5

```
870319967dba4bd02c7a7f8be8ece94f
c9b712f6c347edde22836fb43b927633
75902cf93397d2e2d1797cd115f8347a
```

URL

```
https://helegedada.github.io/test/test
https://api.github.com/repos/helegedada/heihei
http://198.204.231.250/linux-x64
http://198.204.231.250/linux-x86
https://dd.heheda.tk/i.jpg
https://dd.heheda.tk/i.sh
https://dd.heheda.tk/x86_64-static-linux-uclibc.jpg
https://dd.heheda.tk/i686-static-linux-uclibc.jpg
https://dd.cloudappconfig.com/i.jpg
https://dd.cloudappconfig.com/i.sh
https://dd.cloudappconfig.com/x86_64-static-linux-uclibc.jpg
https://dd.cloudappconfig.com/arm-static-linux-uclibcgnueabi.jpg
https://dd.cloudappconfig.com/i686-static-linux-uclibc.jpg
http://d.cloudappconfig.com/i686-w64-mingw32/Satan.exe
http://d.cloudappconfig.com/x86_64-static-linux-uclibc/Satan
http://d.cloudappconfig.com/i686-static-linux-uclibc/Satan
http://d.cloudappconfig.com/arm-static-linux-uclibcgnueabi/Satan
https://d.cloudappconfig.com/mipsel-static-linux-uclibc/Satan
```

C2 Domain

 ```
d.heheda.tk
dd.heheda.tk
c.heheda.tk
d.cloudappconfig.com
dd.cloudappconfig.com
c.cloudappconfig.com
f.cloudappconfig.com
t.cloudappconfig.com
v.cloudappconfig.com
img0.cloudappconfig.com
img1.cloudappconfig.com
img2.cloudappconfig.com
```

IP

```
198.204.231.250     	United States       	ASN 33387           	DataShack, LC       
104.238.151.101     	Japan               	ASN 20473           	Choopa, LLC         
43.224.225.220      	Hong Kong           	ASN 22769           	DDOSING NETWORK    
```

参考链接：[Godlua Backdoor分析报告](https://blog.netlab.360.com/an-analysis-of-godlua-backdoor/)