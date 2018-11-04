# 关联表之前的处理

+  talbe1

| table1_column1 | table1_column2 | table1_column3 |
| --- | --- | --- |
| t1_c1_a | t1_c2_a | t1_c2_a |
| t1_c1_b | t1_c2_b | t1_c2_b |

+ table2

| table2_column1 | table2_column2 | table1_column3 |
| --- | --- | --- |
| t1_c1_a | t2_c2_a | t2_c2_a |
| t1_c1_b | t2_c2_b | t2_c2_b |
| t1_c1_b | t2_c2_c | t2_c2_c |

table1叫做主表
table2叫做子表



# Q 
+ 表A和表B建立关联，那么得出的一张新表的数据过程是怎么表示，又是如何表示他们之间的关联？
+ sql是否是最终分解为dataframe进行处理的。