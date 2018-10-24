# functions 运行原理分析
+ 功能需要自己定义新增列，但是看现在的接口并没有能够满足需求，需要需要深入了解新增列中经常用到的functions的原理

# 函数API
## 时间函数
+ add_months(startDate: Column, numMonths: Int)指定日期添加n月
+ date_add(start: Column, days: Int)指定日期之后n天 e.g. select 
+ date_add('2018-01-01',3)
+ date_sub(start: Column, days: Int)
指定日期之前n天
+ datediff(end: Column, start: Column)
两日期间隔天数
+ current_date()当前日期
+ current_timestamp()当前时间戳，TimestampType类型
+ date_format(dateExpr: Column, format: String)
日期格式化
+ dayofmonth(e: Column)日期在一月中的天数，支持 date/timestamp/string
+ dayofyear(e: Column)日期在一年中的天数， 支持 date/timestamp/string
+ weekofyear(e: Column)日期在一年中的周数， 支持 date/timestamp/string
+ from_unixtime(ut: Column, f: String)
时间戳转字符串格式
+ from_utc_timestamp(ts: Column, tz: String)时间戳转指定时区时间戳
+ to_utc_timestamp(ts: Column, tz: String)
指定时区时间戳转UTF时间戳
+ hour(e: Column)提取小时值
+ minute(e: Column)提取分钟值
+ month(e: Column)提取月份值
+ quarter(e: Column)提取季度
+ second(e: Column)提取秒
+ year(e: Column):提取年
+ last_day(e: Column)指定日期的月末日期
+ months_between(date1: Column, date2: Column)计算两日期差几个月
+ next_day(date: Column, dayOfWeek: String)计算指定日期之后的下一个周一、二...，+ dayOfWeek区分大小写，只接受 "Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"。
+ to_date(e: Column)字段类型转为DateType
+ trunc(date: Column, format: String)日期截断
+ unix_timestamp(s: Column, p: String)指定格式的时间字符串转时间戳
+ unix_timestamp(s: Column)同上，默认格式为 yyyy-MM-dd HH:mm:ss
+ unix_timestamp():当前时间戳(秒),底层实现为unix_timestamp(current_timestamp(), yyyy-MM-dd HH:mm:ss)
+ window(timeColumn: Column, windowDuration: String, slideDuration: String, startTime: String)时间窗口函数，将指定时间(TimestampType)划分到窗口

## 字符串函数
+ ascii(e: Column): 计算第一个字符的ascii码
+ base64(e: Column): base64转码
+ unbase64(e: Column): base64解码
+ concat(exprs: Column*):连接多列字符串
+ concat_ws(sep: String, exprs: Column*):使用sep作为分隔符连接多列字符串 

`
把年和月拼接起来
functions.concat_ws("-",          functions.year(column),season(column))
`
+ decode(value: Column, charset: String): 解码
+ encode(value: Column, charset: String): 转码，charset支持 'US-ASCII', 'ISO-8859-1', 'UTF-8', 'UTF-16BE', 'UTF-16LE', 'UTF-16'。
+ format_number(x: Column, d: Int):格式化'#,###,###.##'形式的字符串
+ format_string(format: String, arguments: Column*): 将arguments按format格式化，格式为printf-style。
+ initcap(e: Column): 单词首字母大写
+ lower(e: Column): 转小写
+ upper(e: Column): 转大写
+ instr(str: Column, substring: String): substring在str中第一次出现的位置
+ length(e: Column): 字符串长度
+ levenshtein(l: Column, r: Column): 计算两个字符串之间的编辑距离（Levenshtein distance）
+ locate(substr: String, str: Column): substring在str中第一次出现的位置，位置编号从1开始，0表示未找到。
+ locate(substr: String, str: Column, pos: Int): 同上，但从pos位置后查找。
+ lpad(str: Column, len: Int, pad: String):字符串左填充。用pad字符填充str的字符串至len长度。有对应的rpad，右填充。
+ ltrim(e: Column):剪掉左边的空格、空白字符，对应有rtrim.
+ ltrim(e: Column, trimString: String):剪掉左边的指定字符,对应有rtrim.
trim(e: Column, trimString: String):剪掉左右两边的指定字符
+ trim(e: Column):剪掉左右两边的空格、空白字符
+ regexp_extract(e: Column, exp: String, groupIdx: Int): 正则提取匹配的组
+ regexp_replace(e: Column, pattern: Column, replacement: Column): 正则替换匹配的部分，这里参数为列。
+ regexp_replace(e: Column, pattern: String, replacement: String): 正则替换匹配的部分
+ repeat(str: Column, n: Int):将str重复n次返回
+ reverse(str: Column): 将str反转
+ soundex(e: Column): 计算桑迪克斯代码（soundex code）PS:用于按英语发音来索引姓名,发音相同但拼写不同的单词，会映射成同一个码。
+ split(str: Column, pattern: String): 用pattern分割str
+ substring(str: Column, pos: Int, len: Int): 在str上截取从pos位置开始长度为len的子字符串。
+ substring_index(str: Column, delim: String, count: Int):Returns the substring from string str before count occurrences of the delimiter delim. If count is positive, everything the left of the final delimiter (counting from left) is returned. If count is negative, every to the right of the final delimiter (counting from the right) is returned. substring_index performs a case-sensitive match when searching for delim.
+ translate(src: Column, matchingString: String, replaceString: String):把src中的matchingString全换成replaceString。

#参考资料
+ 1:[https://liam8.github.io/2018/03/23/spark-sql-functions-api/index.html](https://liam8.github.io/2018/03/23/spark-sql-functions-api/index.html)