# shell-common-function 常用行数
+ `是否有带有参数`
```shell
// --run--
has_arg(){
   if [ $# -lt 2 ];then
     return 0
   fi
   arg_count=$#
   last_arg_index=`expr $arg_count - 1`
   target_arg=${@: -1}
   all_arg=${@:1:$last_arg_index}
   for i in $all_arg;
   do
      if test $target_arg = $i
      then
#        echo "$all_arg has_arg $target_arg"
        return 1
      fi
   done
#   echo "$all_arg has_no_arg $target_arg"
   return 0
}
#has_arg $@ "asa"
has_arg "andre" "asa" "asa"
ret=$?
echo $ret
```
