# scala 对象装java对象

+ RowFactory.create(Object ... values) 转换成为spark.sql的Row
+ JavaConverters java集合和scala集合互相转换
  + list 转seq 
  
  ```
JavaConverters.asScalaIteratorConverter(list.iterator()).asScala().toSeq()
  ```
  
  Pimped Type                            | Conversion Method   | Returned Type
| --- | --- |--- |
scala.collection.Iterator              | asJava              | java.util.Iterator
scala.collection.Iterator              | asJavaEnumeration   | java.util.Enumeration
scala.collection.Iterable              | asJava              | java.lang.Iterable
scala.collection.Iterable              | asJavaCollection    | java.util.Collection
scala.collection.mutable.Buffer        | asJava              | java.util.List
scala.collection.mutable.Seq           | asJava              | java.util.List
scala.collection.Seq                   | asJava              | java.util.List
scala.collection.mutable.Set           | asJava              | java.util.Set
scala.collection.Set                   | asJava              | java.util.Set
scala.collection.mutable.Map           | asJava              | java.util.Map
scala.collection.Map                   | asJava              | java.util.Map
scala.collection.mutable.Map           | asJavaDictionary    | java.util.Dictionary
scala.collection.mutable.ConcurrentMap | asJavaConcurrentMap | java.util.concurrent.ConcurrentMap

Pimped Type                            | Conversion Method   | Returned Type
| --- | --- |--- |
java.util.Iterator                     | asScala             | scala.collection.Iterator
java.util.Enumeration                  | asScala             | scala.collection.Iterator
java.lang.Iterable                     | asScala             | scala.collection.Iterable
java.util.Collection                   | asScala             | scala.collection.Iterable
java.util.List                         | asScala             | scala.collection.mutable.Buffer
java.util.Set                          | asScala             | scala.collection.mutable.Set
java.util.Map                          | asScala             | scala.collection.mutable.Map
java.util.concurrent.ConcurrentMap     | asScala             | scala.collection.mutable.ConcurrentMap
java.util.Dictionary                   | asScala             | scala.collection.mutable.Map
java.util.Properties                   | asScala             | scala.collection.mutable.Map[String, String]


+ `ClassTag$.MODULE$.apply(class)`适配方法前有`[T: ClassTag]`标记的方法如

```
def parallelize[T: ClassTag](
      seq: Seq[T],
      numSlices: Int = defaultParallelism): RDD[T] = withScope {
    assertNotStopped()
    new ParallelCollectionRDD[T](this, seq, numSlices, Map[Int, Seq[String]]())
  }
  
  context.parallelize(seq, context.defaultParallelism(), ClassTag$.MODULE$.apply(String.class))
```

  
#  参考资料
+ [What is the difference between JavaConverters and JavaConversions in Scala?](https://stackoverflow.com/questions/8301947/what-is-the-difference-between-javaconverters-and-javaconversions-in-scala)