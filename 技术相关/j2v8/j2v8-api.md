# j2v8相应的api
V8.add
+ registerJavaMethod(function,"name") 方法联系到类名 注册一个新方法
+ executeScript 执行js脚本
+ executeBooleanScript 返回一个boolean 类型

# 相应的类
+ JavaVoidCallback
+ V8Arrays
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
