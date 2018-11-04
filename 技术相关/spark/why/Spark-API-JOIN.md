# Join api

## 交集合并 (inner)
+ 去掉没有相同key值的记录如

```
tableA
+------+------------+
|column|table1_width|
|row_a|3.0|
|row_b|4.0|
|row_c|5.0|
+------+------------+
tableB
+------+------------+
|column|table2_width|
|row_a|7.0|
|row_c|8.0|
|row_d|9.0|
+------+------------+

tableA.join(tableB)
+------+------------+------------+
|column|table1_width|table2_width|
+------+------------+------------+
| row_a|         3.0|         7.0|
| row_c|         5.0|         8.0|
+------+------------+------------+
```

+ 如果有重复key则是笛卡尔积如

```
tableA
+------+------------+
|column|table1_width|
|row_a|3.0|
|row_a|4.0|
|row_d|6.0|
+------+------------+

tableB
+------+------------+
|column|table2_width|
|row_a|7.0|
|row_d|9.0|
|row_d|10.0|
+------+------------+

tableA.join(tableB)
+------+------------+------------+
|column|table1_width|table2_width|
+------+------------+------------+
| row_a|         3.0|         7.0|
| row_a|         4.0|         7.0|
| row_d|         6.0|         9.0|
| row_d|         6.0|        10.0|
+------+------------+------------+
```

+ 单纯join之后并不是按照顺序来返回来的如

```
table1
| column |  table1_width|
| --- | --- |
|"row_a"| 3.0|
|"row_b"| 4.0|
|"row_c"| 5.0|
|"row_d"| 6.0|

table2

| column |  table2_width|
| --- | --- |
|"row_a"| 7.0|
|"row_b"| 8.0|
|"row_c"| 9.0|
|"row_d"| 10.0|
   
   table1.join(table2,"column")
   得到的可能是
                       
| column |  table1_width|  table2_width|
| --- | --- |--- |
|"row_a"| 3.0|7.0|
|"row_c"| 5.0|9.0|
|"row_b"| 4.0|8.0|
|"row_d"| 6.0|10.0|
从这里就可以知道其实spark 中单纯的join之后并不会保持左表的顺序
```

+ 不同列的等于

```
Dataset<Row> join = dataFrame1.join(dataFrame2,
                                            dataFrame1.col("table1_column")
                                                    .equalTo(dataFrame2.col("table2_column")));
                                                    
+-------------+------------+-------------+------------+
|table1_column|table1_width|table2_column|table2_width|
+-------------+------------+-------------+------------+
|        row_a|         3.0|        row_a|         7.0|
|        row_a|         4.0|        row_a|         7.0|
|        row_d|         6.0|        row_d|         9.0|
|        row_d|         6.0|        row_d|        10.0|
+-------------+------------+-------------+------------+

+-------------+------------+
|table1_column|table1_width|
|row_a|3.0|
|row_a|4.0|
|row_d|6.0|
+-------------+------------+

+-------------+------------+
|table2_column|table2_width|
|row_a|7.0|
|row_d|9.0|
|row_d|10.0|
+-------------+------------+                                        
```

+ 不同列等于且合并成同一列

## 并集合并(outer)
+ 规则和交集合并一样，不同点在于取的是两个数据集的合集，即如果数据集里面有的数据在结果数据集里面必然存在，左右合并可以说是其子集。

```
Dataset<Row> join = dataFrame1.join(dataFrame2,
                                            dataFrame1.col("column").equalTo(dataFram2.col("column")),"outer");
                                            
+------+------------+------+------------+
|column|table1_width|column|table2_width|
+------+------------+------+------------+
| row_a|         3.0| row_a|         7.0|
| row_a|         3.0| row_a|         8.0|
| row_a|         4.0| row_a|         7.0|
| row_a|         4.0| row_a|         8.0|
| row_c|         5.0| row_c|         8.0|
| row_b|         4.0|  null|        null|
|  null|        null| row_d|         9.0|
+------+------------+------+------------+

+------+------------+
|column|table1_width|
|row_a|3.0|
|row_a|4.0|
|row_b|4.0|
|row_c|5.0|
+------+------------+

+------+------------+
|column|table2_width|
|row_a|7.0|
|row_a|8.0|
|row_c|8.0|
|row_d|9.0|
+------+------------+
```

## 左合并(left)
大致同outer
## 右合并(outer)
大致同outer

# Q 

# 参考资料
+ 1:[https://blog.csdn.net/u011239443/article/details/54377074](https://blog.csdn.net/u011239443/article/details/54377074)
+ 2:[https://www.oreilly.com/library/view/data-algorithms/9781491906170/ch04.html](https://www.oreilly.com/library/view/data-algorithms/9781491906170/ch04.html)
+ 3:[多表操作 join https://blog.csdn.net/coding_hello/article/details/75452436](https://blog.csdn.net/coding_hello/article/details/75452436)
