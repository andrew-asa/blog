# shell-判断
参数|	说明
:---| :---
-eq	| 等于
-ne	| 不等于
-gt	| 大于
-ge	| 大于等于
-lt	| 小于
-le	| 小于
-z  | 空串
=   | 两个字符相等
!=  | 两个字符不等
-n  | 非空串
-e  |	如果文件存在则为真
-r  |	如果文件存在且可读则为真
-w  |	如果文件存在且可写则为真
-x  |	如果文件存在且可执行则为真
-s  |	如果文件存在且至少有一个字符则为真
-d  |	如果文件存在且为目录则为真
-f  |	如果文件存在且为普通文件则为真
-c  |	如果文件存在且为字符型特殊文件则为真
-b  |	如果文件存在且为块特殊文件则为真

```
if [ $a -eq $b ]
    then
        echo "$a -eq $b : a 等于 b"
    else
       echo "$a -eq $b : a 等于 b"
fi
```


```
num1="ru1noob"
num2="runoob"
if test $num1 = $num2
then
    echo '两个字符串相等!'
else
    echo '两个字符串不相等!'
fi
```
```
cd /bin
if test -e ./bash
then
    echo '文件已存在!'
else
    echo '文件不存在!'
fi
```