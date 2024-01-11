原题
```php
<?php
header("Content-type:text/html;charset=utf-8");
error_reporting(0);
include 'flag.php';
$b='ssAEDsssss';    //定义变量b
extract($_GET);     //extract
if(isset($a)){  
    $c=trim(file_get_contents($b));
    if($a==$c){
        echo $myFlag;
    }else{
        echo'继续努力，相信flag离你不远了';
    }
}
?>
```
这是储存在服务器上的脚本。

`extract()` 函数从数组中将变量导入到当前的符号表。

该函数使用数组键名作为变量名，使用数组键值作为变量值。针对数组中的每个元素，将在当前符号表中创建对应的一个变量。

在 PHP 中，预定义的 `$_GET` 变量用于收集来自 `method="get"` 的表单中的值。

本题具体方法为访问 http://124.16.75.117:51001?a= 即可。通过这样传参我们保证 `isset($a)` 的值是 `True` 而服务器上又没有名为 ssAEDsssss 的文件或是以此文件为名的文件为空，因此可以直接输出 flag。如果服务器上有以 b 为名字的文件，则访问 http://124.16.75.117:51001?a=&b= 即可（不存在名字为空的文件，因此 `file_get_contents($b)` 此时一定是空的）。