# redis 相关命令

## 常用命令整理
### 字符串
+ 字符串不仅可以存放文本，还可以存放数值，浮点数，如果存放的字符串可以被解释成为十进制的数值或者浮点数则可可以使用数值操作的命令
+ 字符串的含义
  + 没有规律的字符串
  + 数值
  + 二进制

+ 数值常用命令

命令| 说明 | 返回值
--- | ---| ---
incr key | 数值+1 | +1 结果
decr | 数值-1 | -1结果
incrby key value| 数值+value | +value 结果
decrby key value | 数值-value | -value结果
decrbyfloat key fvalue | 数值+精度值value | -精度值value结果


+ 字符串常用命令

命令| 说明 | 返回值
--- | ---| ---
set key value | 设置一个key，value | true|false
del key1,key2 | 删除一个key | 删除成功的个数
get key | 获取一个key的value值 | 没有返回nil
append key value| 字符串后面添加 | 字符串长度
getrange key start end| 获取从start到end的字符串，包括end | 获取的字符串
getbit key offset| 字符串看做是二进制字符串，获取第offset位的二进制值 | 获取的字符串

+ 二进制常用命令

命令| 说明 | 返回值
--- | ---| ---
getbit key offset| 字符串看做是二进制字符串，获取第offset位的二进制值 | 获取的字符串

### 列表

### 集合

### 散列

### 有序集合

### 公共
命令| 说明 | 返回值
--- | ---| ---
ping | 检测服务器端是否已经开启 | pong(成功)或者错误信息
info | 当前redis服务器信息 | 当前redis服务器信息