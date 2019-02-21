## 常用shell命令收集
+ 获取某个文件夹中指定文件类型文件的文件行数
` find . "(" -name "*.java" ")"  -print | grep -v test | xargs wc -l | grep total `

+ 获取文件夹下指定后缀名的文件个数
` find . "(" -name "*.wav" ")" `

+ 计算文件md5值 `md5 file`
+ 计算文件base64 `base64 file`
+ 计算字符串base64 `echo xxx base64`
+ 杀死占用某端口的所有进程 `kill -9 $(l:进程号 -t)`
+ !n 执行第n个命名
+ !! 执行上一条命令
+ !command 执行上一条一command开头的命令

##  常用bash shell 快捷键

### 删除
+ ctrl+u: 删除光标左边所有
+ ctrl+k: 删除光标右边所有
+ ctrl+l: 清屏

### 移动
+ ctrl+b: 前移一个字符(backward)
+ ctrl+f: 后移一个字符(forward)
+ alt+b: 前移一个单词
+ alt+f: 后移一个单词
+ ctrl+a: 移到行首（a是首字母）
+ ctrl+e: 移到行尾（end）
+ ctrl+xx: 行首到当前光标替换

### zsh
+ d: 列出以前的打开的命令
+ j: jump到以前某个目录，模糊匹配

## 命令专题
+ awk
    + 截取第n个空格符隔开的信息 `awk '{print $n}'`
      例如 "hello world, are you ok" 像获取`you`可以 awk '{print $4}'
      

## 参考资料
+ [Bash Shell常用快捷键 https://github.com/hokein/Wiki/wiki/Bash-Shell%E5%B8%B8%E7%94%A8%E5%BF%AB%E6%8D%B7%E9%94%AE](https://github.com/hokein/Wiki/wiki/Bash-Shell%E5%B8%B8%E7%94%A8%E5%BF%AB%E6%8D%B7%E9%94%AE)

      
      
      

      
      
      
      
      
      