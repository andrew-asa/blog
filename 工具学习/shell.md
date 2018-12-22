### 常用shell命令收集
+ 获取某个文件夹中指定文件类型文件的文件行数
` find . "(" -name "*.java" ")"  -print | grep -v test | xargs wc -l | grep total `

+ 获取文件夹下指定后缀名的文件个数
` find . "(" -name "*.wav" ")" `