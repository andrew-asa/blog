# 零散
+ 不同的驱动有不同的参数，真正重要的并不是了解不同驱动的参数，而是了解整个驱动加载的机制。
+ 可以把spark理解成为一个bi，首先需要进行配置参数，然后才可以使用，可以把Dataset的操作当成是分析表的操作。

# Q
+ 一个SparkSession能用多久？
+ SparkSession.master 怎么使用？

# Guess
+ 定制一个RelationProvider，可以理解为表的驱动
驱动里面可以返回Schma信息，表示表的信息，然后定制一个RDD,RDD里面的compute


 ```sequence
Title:自定义数据集
reader.load->relation.buildScan: 调用
relation.buildScan->RDD: 生成
RDD->RDD.compute: 返回用户的数据
 ```