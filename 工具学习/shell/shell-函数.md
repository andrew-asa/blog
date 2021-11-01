# shell-function
## 不带参函数
```shell
// --run--
demoFun(){
    echo "这是我的第一个 shell 函数!"
}
demoFun
```

## 带参函数调用
```shell
// --run--
funWithParam(){
    echo "第一个参数为 $1 !"
    echo "第二个参数为 $2 !"
    echo "第十个参数为 $10 !"
    echo "第十个参数为 ${10} !"
    echo "第十一个参数为 ${11} !"
    echo "参数总数有 $# 个!"
    echo "作为一个字符串输出所有参数 $* !"
}
funWithParam 1 2 3 4 5 6 7 8 9 34 73
```
`
注意，$10 不能获取第十个参数，获取第十个参数需要${10}。
当n>=10时，需要使用${n}来获取参数。
`
```sh
// --run--
function test(){
    echo "test here"
    return 100
}
ret=`test`
echo "return: $?"
echo "ret: $ret"
```
```sh
// --run--
function test(){
    echo 100
    return 100
}
ret=`test`
echo "return: $?"
echo "ret: $ret"
if [ $ret -eq 100 ]
    then
    echo "ret equal 100"
fi
```
`注意：函数调用返回值需要用 ret=$?进行赋予`





