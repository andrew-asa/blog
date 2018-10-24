# DataFrame
+ registerTempTable("tableName") 注册一张临时表
+ createOrReplaceTempView("tableName") 将表加载入spark-sql，后面就可以运用spark-sql进行相关的操作
+ drop 删掉某一行
+ filter(Column column)
   + Column 可以用dataSet.col(columnName)进行获取
+ filter(String conditionExpr)
+ filter(FilterFunction func)
+ filter(func: T => Boolean)

+ 新增列的多种方式
   + 新增一个表达式(spark 中支持的函数 参考资料:1)
   + mapPartitions 可以自定义运算
   `        dataSet=dataSet.selectExpr("*", expression + " as " + columnName);
  `
+ dataset.withColumn(String colName,Column column)
+ 自定义函数列（遇到自定义的函数只能先落地？）

+ mapPartitions
+ join

+ 动作
    + collect
    + collectAsList 已经是获取数据了

+ 应用方案
   + 过滤
       + 列属于多个值过滤
          + 方案一 column.equalTo + column.or 拼接
          + 方案二 sql 拼接in过滤条件

# RDD

# Column
## 获取
+ dataFrame.apply
## 用法
+ cast 转换成什么类型
    + 注意data类型不能直接转成long类型，需要转换成TimestampType再转换成为longendColumn.cast(DataTypes.TimestampType).cast(DataTypes.LongType)
+ 
+ 


# J
+ 得到一个DataSet<Row> 实际上相当于得到一张表的概念，表的获取理论上来说可以从不同的数据源中产生。然后可以通过注入进入spark-sql 中，或者说session中后面就可以像数据库那样进行sql操作。
+ registerTempTable和createOrReplaceTempView又有什么区别。这些表又是存放到哪里，以及生命周期又是怎样的？

# Partition
+ 分区的概念


# 参考资料
+ 1:[https://liam8.github.io/2018/03/23/spark-sql-functions-api/index.html](https://liam8.github.io/2018/03/23/spark-sql-functions-api/index.html)
+ 2:[http://www.cnblogs.com/CareySon/p/3411176.html](http://www.cnblogs.com/CareySon/p/3411176.html)