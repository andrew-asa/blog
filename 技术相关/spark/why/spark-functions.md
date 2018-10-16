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

#参考资料
+ 1:[https://liam8.github.io/2018/03/23/spark-sql-functions-api/index.html](https://liam8.github.io/2018/03/23/spark-sql-functions-api/index.html)