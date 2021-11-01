### 零散知识点
+ 单例的正确写法

 ```
 public static Class getInstance() {

        if (INSTANCE == null) {
            synchronized (Class.class) {
                if (INSTANCE == null) {
                    INSTANCE = new Class();
                }
            }
        }
        return INSTANCE;
    }
 ```
 
+ 执行jar中的某个类
   + `java -cp xxx.jar xxx.com.xxxx ` 它会找到这个类的main函数，开始执行.
     其中-cp命令是将xxx.jar加入到classpath，这样java class loader就会在这里面查找匹配的类。
     
+ 输入当前线程id`Thread.currentThread().getId() `

### 遗留问题
+ window上面jvm引用文件不能删除？





### 参考资料
+ 1: [java设计模式大全源码+注释+说明](https://github.com/iluwatar/java-design-patterns)
+ 2:[OpenJDK 源码下载](http://hg.openjdk.java.net/jdk8/jdk8)