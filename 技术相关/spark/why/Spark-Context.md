# session context 相关
# 零散笔记
+ 内存数据转Rdd
+ makeRDD
+ parallelize
+ RDD转DataSet
+ JavaSparkContext和一般的SparkContext的区别


# Q
+ 如何查询context中一共共存了多少张表？
+ 临时表信息？
+ 临时表的生存时机？
   A: session 存在的就会存在
+ 如何进行释放临时表？
+ 所有表都本地化如何进行？
+ 表关联和join的区别
   join是两张表之间的有关联字段的笛卡尔积
   关联得到的数据却是以大表为主
+ 如何判断一张表是否已经存在context里面

# 参考资料
+ [spark源码阅读笔记RDD（七） RDD的创建、读取和保存](https://blog.csdn.net/legotime/article/details/51329692)