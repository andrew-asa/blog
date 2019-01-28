# redis 相关命令

## 常用命令整理
### 文本
命令| 说明 | 返回值
--- | ---| ---
set key value | 设置一个key，value | true|false
del key1,key2 | 删除一个key | 删除成功的个数
get key | 获取一个key的value值 | 没有返回nil

### 列表

### 集合

### 散列

### 有序集合

### 公共
命令| 说明 | 返回值
--- | ---| ---
ping | 检测服务器端是否已经开启 | pong(成功)或者错误信息
info | 当前redis服务器信息 | 当前redis服务器信息