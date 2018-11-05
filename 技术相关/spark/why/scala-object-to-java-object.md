# scala 对象装java对象

+ RowFactory.create(Object ... values) 转换成为spark.sql的Row
+ JavaConverters 
  + list 转seq 
  
  ```
JavaConverters.asScalaIteratorConverter(list.iterator()).asScala().toSeq()
  ```