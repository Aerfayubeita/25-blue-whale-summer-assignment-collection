# [Dest0g3 520迎新赛]EasyPHP

## 题目来源

[Dest0g3 520迎新赛]EasyPHP

## 题目方向 + 知识点

PHP 源码审计、`set_error_handler()`、匿名函数 `use` 捕获变量、引用传递、数组触发报错、`Array to string conversion`。

## 解题思路

这题源码如下：

```php
<?php
highlight_file(__FILE__);
include "fl4g.php";
$dest0g3 = $_POST['ctf'];
$time = date("H");
$timme = date("d");
$timmme = date("i");
if(($time > "24") or ($timme > "31") or ($timmme > "60")){
    echo $fl4g;
}else{
    echo "Try harder!";
}
set_error_handler(
    function() use(&$fl4g) {
        print $fl4g;
    }
);
$fl4g .= $dest0g3;
?>
```

这题第一眼很容易被前面的时间判断吸引：

```php
if(($time > "24") or ($timme > "31") or ($timmme > "60")){
    echo $fl4g;
}else{
    echo "Try harder!";
}
```

但实际上这里根本走不进输出 flag 的分支。

因为：

- `date("H")` 的取值范围是 `00-23`
- `date("d")` 的取值范围是 `01-31`
- `date("i")` 的取值范围是 `00-59`

所以：

- 小时不可能大于 `24`
- 日期不可能大于 `31`
- 分钟不可能大于 `60`

也就是说，这个条件永远不成立，程序正常情况下只会输出：

```text
Try harder!
```

真正的利用点其实在下面这段：

```php
set_error_handler(
    function() use(&$fl4g) {
        print $fl4g;
    }
);
$fl4g .= $dest0g3;
```

先看 `set_error_handler()`。这个函数的作用是注册一个 **自定义错误处理函数**。也就是说，一旦后面的代码触发了 PHP 报错，程序就会执行这里定义的匿名函数：

```php
function() use(&$fl4g) {
    print $fl4g;
}
```

这里的 `use(&$fl4g)` 也非常关键。它表示这个匿名函数要使用外部变量 `$fl4g`，并且是 **按引用捕获**。所以后面如果 `$fl4g` 的值发生变化，这个报错函数里看到的也是变化后的真实值。

接着看最后一行：

```php
$fl4g .= $dest0g3;
```

这句的意思是把 `$dest0g3` 拼接到 `$fl4g` 后面。正常情况下如果 `$dest0g3` 是字符串，就不会出问题。

但题目里：

```php
$dest0g3 = $_POST['ctf'];
```

也就是说，`$dest0g3` 完全由我们控制。那我们就不传普通字符串，而是故意把 `ctf` 传成 **数组**。

例如：

```text
ctf[]=1
```

这样一来：

```php
$_POST['ctf']
```

得到的就不是字符串，而是数组。此时最后一行变成：

```php
$fl4g .= 数组
```

PHP 在字符串拼接数组时会触发报错，典型报错信息就是：

```text
Array to string conversion
```

而这个错误一旦出现，就正好会触发前面注册好的错误处理函数：

```php
function() use(&$fl4g) {
    print $fl4g;
}
```

于是 `$fl4g` 就会被打印出来，flag 也就拿到了。





## Flag

```text
ctf[]=1
```

即可触发报错并输出 flag。