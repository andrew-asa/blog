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
+ 集合元素可以重复
+ 常用命令

命令| 说明 | 返回值
--- | ---| ---
rpush key value1 value2 | 将一个或多个值推入列表的右端 | 列表元素个数
lpush key value1 value2 | 将一个或多个值推入列表的左端 | 列表元素个数
rpop key  | 移除定返回最右端元素 | 最右端元素
lpop key  | 移除定返回最左端元素 | 最左端元素
lindex key  offset| 返回列表中偏移量为offset的元素 | 
lrange key  star end| 获取 start<= index <= end | 
ltrim key  star end| 只保留 start<= index <= end | 

+ 上面的操作命令都是非阻塞的，没有则返回nil，是否说redis非阻塞的命令都有一个准则即：所有元素返回操作没有都是以nil代替

命令| 说明 | 返回值
--- | ---| ---
blpop key key2 timeout | 从第一个非空列表中弹出位于最左端的元素或者再timeout 秒内阻塞等待可弹出的元素出现 | 


### 集合
+ 元素不重复的集合
+ 可以看成是java中的set
+ 常用命令

命令| 说明 | 返回值
--- | ---| ---
sdiff key key2 ... | 返回存在于第一个集合，但不存在于其他集合中的原始 | 数学上的差集运算

### 散列
+ key value 对，可以看成是一个小redis，一个值也可以看成是关系型数据库中的一行
+ hash key 值不重复，如果重复后面的会覆盖前面的
+ 常用命令

命令| 说明 | 返回值
--- | ---| ---
hmset key hashkey hashvalue hashkey1 hashvalue1 ... | 为散列里面的一个或过个key设置值 | 
hmget key hashkey hashkey1 ... | 获取散列里面的一个或多个key值 | 
hlen key | 获取散列里面键值对数量 | 
hdel key hashkey1 hashkey2 ... | 删除一个或多个键值对 | 

### 有序集合

+ 可以类比Map<String,Number> 类型
+ 常用命令

命令| 说明 | 返回值
--- | ---| ---
zadd key score member [key score member ...] | 将带有给定分值的成员添加到有序集合里面 | 成功添加数量
zrm key member [member ...]| 删除制定成员| 成功删除的数量
zcard key | 有序集合数量 | 

### 发布与订阅

### 公共
命令| 说明 | 返回值
--- | ---| ---
ping | 检测服务器端是否已经开启 | pong(成功)或者错误信息
info | 当前redis服务器信息 | 当前redis服务器信息
bgsave | 创建快照（fork线程来完成这件事情） | 
save time condition | 间隔time时间，当这期间进行了codition次写操作则备份文件 | 
keys *  | 列出匹配正在的key列表（阻塞行命令） | 
exits key  | 测试key值是否存在 | 
type key  | key的类型 | 
expire key  | 设置key值过期时间（s） | 
ttl key  | 剩余多长时间过期（s） | -1表示已经过期

### 事务

+ 一般的关系型数据库select for update时采用的是悲观锁策略，即当开启事务的时候进行枷锁，其它修改操作需要等待锁的完成
+ redis采用的是乐观锁策略。即不进行锁定，所以需要程序自己进行改变的处理。
+ 常用命令

命令| 说明 | 返回值
--- | ---| ---
multi | 开启一个事务（然后需要依次写入命令） | 
exec | 执行提交的事务里面的命令 | 
watch | 观察在到执行命令之前这段时间数据是否有改变 | 
enwatch | 取消观察 | 
discard | 取消观察并丢弃所有已经放到事务中的命令 | 

### 参考资料
+ [redis 命令 http://www.redis.cn/commands.html#list](http://www.redis.cn/commands.html#list)