# [ACTF2020 新生赛]BackupFile

## 题目来源

[ACTF2020 新生赛]BackupFile

## 题目方向 + 知识点

源码泄露、备份文件发现、PHP 弱类型比较、`is_numeric()`、`intval()`、`==` 非严格比较。

## 解题思路

这道题打开首页之后，页面只显示一句话：

```text
Try to find out source file!
```

这种提示其实已经很直接了，意思就是不要在首页硬猜参数，而是先去找源码文件或者备份文件。

所以我第一步没有急着 fuzz 参数，而是先去测常见的源码泄露文件名，比如：

- `index.phps`
- `index.php~`
- `index.php.bak`
- `www.zip`

结果访问：

```text
/index.php.bak
```

成功拿到了源码：

```php
<?php
include_once "flag.php";

if(isset($_GET['key'])) {
    $key = $_GET['key'];
    if(!is_numeric($key)) {
        exit("Just num!");
    }
    $key = intval($key);
    $str = "123ffwsfwefwf24r2f32ir23jrw923rskfjwtsw54w3";
    if($key == $str) {
        echo $flag;
    }
}
else {
    echo "Try to find out source file!";
}
```

源码一出来，逻辑就非常清楚了。

程序要求传一个 `key` 参数，并且先经过：

```php
if(!is_numeric($key)) {
    exit("Just num!");
}
```

这意味着输入必须是“数字形式”，否则程序直接退出。

之后程序又做了：

```php
$key = intval($key);
```

也就是说，不管我传的是：

```text
123
0123
123.0
```

最后都会被转成整数 `123`。

真正的关键在最后这句：

```php
if($key == $str)
```

注意这里用的是 **`==`**，不是 **`===`**。

而 `$str` 的值是：

```php
"123ffwsfwefwf24r2f32ir23jrw923rskfjwtsw54w3"
```

这是一个“以数字开头的字符串”。在 PHP 的弱类型比较里，当一个整数和一个以数字开头的字符串使用 `==` 比较时，PHP 会把这个字符串按数值语义去解释，前面的数字部分会被拿来参与比较。

也就是说这里相当于：

```php
123 == "123ffwsfwefwf24r2f32ir23jrw923rskfjwtsw54w3"
```

在这种弱比较场景下，右边字符串会被当成数字 `123` 来参与比较，因此条件成立。

于是这题就根本不需要什么复杂构造，直接传：

```text
?key=123
```

就可以拿到 flag。



## Flag

`CTF2{248aca9b-a0dc-4d06-bc6a-b4e2f751e65b}`