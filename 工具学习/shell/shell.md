### 常用shell命令收集
+ 获取某个文件夹中指定文件类型文件的文件行数
` find . "(" -name "*.java" ")"  -print | grep -v test | xargs wc -l | grep total `

+ 获取文件夹下指定后缀名的文件个数
` find . "(" -name "*.wav" ")" `

+ 计算文件md5值 `md5 file`
+ 计算文件base64 `base64 file`
+ 计算字符串base64 `echo xxx base64`
+ 杀死占用某端口的所有进程 `kill -9 $(l:进程号 -t)`