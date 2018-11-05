# java 调用scala接口中的一些解释

# 零散笔记
+ Seq做为参数的使用 

```
def makeRDD[T: ClassTag](
      seq: Seq[T],
      numSlices: Int = defaultParallelism): RDD[T] = withScope {
    parallelize(seq, numSlices)
  }
```