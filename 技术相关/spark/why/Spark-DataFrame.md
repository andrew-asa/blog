# 零散
+ 不同的驱动有不同的参数，真正重要的并不是了解不同驱动的参数，而是了解整个驱动加载的机制。
+ 可以把spark理解成为一个bi，首先需要进行配置参数，然后才可以使用，可以把Dataset的操作当成是分析表的操作。
+ 涉及到shuffle的操作的类其类和子类都需要实现序列化接口

# Q
+ 一个SparkSession能用多久？
+ SparkSession.master 怎么使用？

# Guess
+ 定制一个RelationProvider，可以理解为表的驱动
驱动里面可以返回Schma信息，表示表的信息，然后定制一个RDD,RDD里面的compute


 ```sequence
Title:自定义数据集
reader->RelationProvider:format(类名)决定使用哪个驱动
RelationProvider->reader:使用option中提供的参数生成相应的relation
reader.load->relation.schema:调用
relation.schema->reader.load:结合schema生成Dataset<Row>
Dataset.show->relation.buildScan:生成RDD<Row>
 ```
 + 但是RDD是如何转换成为DataSet<Row>的呢？
 + BaseRDD<InternalRow> 迭代器里面命名要求的是返回InternalRow但是在真正进行实习的实现的时候要求的竟然是Row类型？scala语言是否是像js那样的弱语言类型？
 