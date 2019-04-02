# 基于redis实现的集群集合

## 实现点
### map
+ 与redis的散列对应
+ key，value先进行序列化然后再进行存放到redis里面。
+ 单机情况下集合的操作如map 的remove(key,old) 为了确保原子性会先把对象锁住然后获取old的值在比较是是否相同，但是在使用jedis实现集群map的情况不能这样处理，又因为redis能够确保lua脚本执行的时候能够确保原子性，所以可以把在单机中的集合操作代码搬到lua脚本中。
 ```
 @Override
    public boolean remove(Object key, Object value) {

        String script = "if redis.call('hget', KEYS[1], ARGV[1]) == ARGV[2] then "
                + "return redis.call('hdel', KEYS[1], ARGV[1]) "
                + "else "
                + "return 0 "
                + "end";
        Object o = redisClient.eval(encodeString(script),
                                    1,
                                    nameBytes,
                                    encodeMapKey(key),
                                    encodeMapValue(value));
        return RET_SUCCESS_OBJ.equals(o);
    }
 ```
 
### CacheMap 支持最大缓存时间，最大空闲时间特性的map
+   timeToIdleSeconds： 设定允许对象处于空闲状态的最长时间，以秒为单位。当对象自从最近一次被访问后，如果处于空闲状态的时间超过了timeToIdleSeconds属性值，这个对象就会过期。
+   timeToLiveSeconds：设定对象允许存在于缓存中的最长时间，以秒为单位。当对象自从被存放到缓存中后，如果处于缓存中的时间超过了 timeToLiveSeconds属性值，这个对象就会过期。
+ 在redisson中remainTimeToLive方法没有过期时返回的是一个负数，但是，按照接口的解释明显不是，所以应该是redisson的一个bug。
 
### Set
+ 与redis的集合对象对应
+ 需要注意removeAll方法表明，如果set的元素改变才返回true，如果没有改变返回的是false，所以removeAll 一个empty collection的时候返回的是false。
+ iterator() 方法的实现
   + 所有集群基本数据结构的迭代器的实现都是一个坑点。
   + 直接用scan类命令，当返回的下一次迭代的索引不为"0"字符串时则一直迭代下去。
   + 迭代器里面的remove方法直接调用map set的remove方法真正进行删除元素。

### Queue
+ 直接继承RedisList
+ element和peek方法都是通过get(0)方法实现，不过区别在于element如果get出来是null抛一个NoSuchElementException异常（接口规定）
+ remove和poll方法都是直接调用lpop命令，不同的是remove没有返回值则抛一个NoSuchElementException异常（接口规定）

### List
+ 与redis中的列表对应
+ 像remove(int index) index 参数是一个整数，不能像map那样简单的对其进行序列化作为一个参数传给evl函数，而应该是先把index转换成字符串然后getBytes作为参数。
  `new Integer(index).toString().getBytes()`
+ subList方法的解释
    + 1，该方法返回的是父list的一个视图，从fromIndex（包含），到toIndex（不包含）。fromIndex=toIndex 表示子list为空
   + 2，父子list做的非结构性修改（non-structural changes）都会影响到彼此：所谓的“非结构性修改”，是指不涉及到list的大小改变的修改。相反，结构性修改，指改变了list大小的修改。
  + 3，对于结构性修改，子list的所有操作都会反映到父list上。但父list的修改将会导致返回的子list失效。
          + ArraryList的实现是子list 保存一个modCount，parent list 也是一样，子list的结构性修改其实就是调用父类的相应的方法，同时保存父类的modCount，每次修改前先判断父子list的modCount变量是否相同，如果不相同就说明父类已经发生了结构性的修改。这时直接进行报错。
          
    ```
    private void checkForComodification() {
            if (ArrayList.this.modCount != this.modCount)
                throw new ConcurrentModificationException();
        }
    ```
    + 但是对于集群来说这是不现实的。
    +  ListIterator.set 
        +  但是ListIterator可以实现对象的修改，set()方法可以实现
        + 替换它访问过的最后一个元素
        + 当且仅当next,previous已经调用时才能进行操作
    +  ListIterator.add
       + 在next()方法返回的元素之前或previous()方法返回的元素之后插入一个元素.
       + 同时如果add之后不能马上进行set操作
    

## 优化

## Q & A
+ 迭代器的行为？迭代器是返回获取时刻所拥有的对象构成的迭代器还是允许迭代器能够在其没有迭代完成时随着集合的操作动态变化?


## 性能对比
### map
方法名 | 参数 |redis耗时 | 单机耗时
--- | ---| ---|---
putAll | 整数 1 ~ 100000 | 1041 ms | 394 ms

#### 遗留问题
+ putAll 个数有限制1000000（整数） 报错
+ key，value 序列化快慢的比较，方案确定


## 参考资料
+ [JAVA中ListIterator和Iterator详解与辨析 https://blog.csdn.net/longshengguoji/article/details/41551491](https://blog.csdn.net/longshengguoji/article/details/41551491)
+ [java队列——queue详细分析 https://www.cnblogs.com/lemon-flm/p/7877898.html](https://www.cnblogs.com/lemon-flm/p/7877898.html)
