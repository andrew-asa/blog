# shell 常用命令
+ du -hs dir 显示某个文件夹占用空间大小
+ ps axw -o pid,ppid,user,%cpu,vsz,wchan,command | egrep '(redis|PID)'

## find 
+ find 文件路径 参数
    + find ~ -iname  "screen*"    比如你可以通过以下命令在用户文件夹中搜索名字中包含screen的文件