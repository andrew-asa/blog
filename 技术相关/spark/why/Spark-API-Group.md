# 零散笔记
+ dataSet.groupBy得到RelationalGroupedDatas之后本身就支持最大值，最小值，平均值，记录数的常用统计函数，但是美中不足的是需要一些技巧来定制这些汇总列的名字
+ 同join一样，分组汇总可能是涉及到shuffle操作所以记录的排序顺序并没有按照之前的顺序

# 功能点
+ 求和
+ 求平均
+ 求最大
+ 求最小
+ 标准差
+ 方差
+ 去重计数
+ 记录个数
+ 同期
+ 环期
+ 同比
+ 环比
    + 应该可以用udaf来进行获取

# 碰到的问题
+ 有两列只对一列进行groupBy然后agg会产生java.lang.ClassCastException: java.lang.String cannot be cast to java.lang.Double 异常

# Q
+ dataSet.groupBy 出来的是 RelationalGroupedDatas 这个类想表示的是什么
+ 去重计数和记录个数的区别，分组完之后不久等同于已经去重？
+ 从groupBy之后可以指定聚合函数那么是否可以自定义自己的聚合函数，像中位数这种函数是否可以？
+ 如果我仅仅想分组不需要汇总这个应该怎么表示？

# 参考资料
+ 1:[Spark DataFrame 使用UDF实现UDAF的一种方法 https://segmentfault.com/a/1190000014088377](https://segmentfault.com/a/1190000014088377)