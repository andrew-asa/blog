# j2v8相应的api
V8.add
+ registerJavaMethod(function,"name") 方法联系到类名 注册一个新方法
+ executeScript 执行js脚本
+ executeBooleanScript 返回一个boolean 类型
+ v8会对对象进行自动释放

# 相应的类
+ JavaVoidCallback
+ V8Arrays
   + getType("attr")获取参数类型，类型定义在V8Value例如：如果参数是一个方法则getType返回V8_FUNCTION
+ V8Object
    + add 绑定一个成员
    ```
v8Object.add("width", canvas.getWidth());
```
+ setPrototype 设置原形？？

# Q
+ 什么时候需要释放资源，又需要释放什么资源?

# 参考资料
+ 1: [https://www.heqiangfly.com/2017/08/10/open-source-j2v8-registerting-java-callback/](https://www.heqiangfly.com/2017/08/10/open-source-j2v8-registerting-java-callback/)
