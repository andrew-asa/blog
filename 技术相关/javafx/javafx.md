### 零散知识点
+ 窗口居中 先show再进行调用

```
private void setWindowCenter(Stage stage) {

        Rectangle2D primaryScreenBounds = Screen.getPrimary().getVisualBounds();
        stage.setX(primaryScreenBounds.getMinX() + (primaryScreenBounds.getWidth() - stage.getWidth()) / 2.0);
        stage.setY(primaryScreenBounds.getMinY() + (primaryScreenBounds.getHeight() - stage.getHeight()) / 2.0);
    }
```

+ 应用fxml里面定义的元素

``` 
现在fxml中定义样式，控件，然后再程序里面进行调用
```
+ Stage
+ 
    + 设置应用程序图标 `stage.getIcons().add(new Image("file:xxx.png"));`

+ TextField
   + 圆角
   + 放置搜索图表
   + 字体大小
   
   		```
   		setStyle(" -fx-font: normal 16px \"Helvetica Neue\", Arial, \"PingFang SC\", \"Hiragino Sans GB\", \"Microsoft YaHei\", Heiti;font: normal 12px \"Helvetica Neue\", Arial, \"PingFang SC\", \"Hiragino Sans GB\", \"Microsoft YaHei\", Heiti;")
   		```
   		但是设置出来的字体并不能像css那样进行配置
   
   + 水印
   + 事件
      + setOnKeyPressed 按键事件

+ 字体
    + 创建字体
    ```       
     Font font =  Font.font(family, size); 
    ```
+ Scene
    + 设置背景透明 `setFill(null)`
+ AnchorPane

+ VBox 

   ``
   单列布局
   ``
   
+ ListView `listview能够解决动态添加元素的机制`
    + 禁止滚动条
    + 动态高度
      
	
	
	
+ JavaFX Scene Builder
    + FXMLLoader.load `反序列化，在Scene Builder里面先完成控件，界面的布局设置，然后调用loader出来进行反序列化成对应的布局，控件`

``UI设置与代码分离，等同于chrome的控制台？``

### 观点
+ 一个jvm只能运行一个javafx程序

```
为什么，是什么做了限制？
```

+ 创建和关闭Stage只能通过javafx提供的线程里面进行创建

```
为什么，为什么要做这样的限制，如果不进行这样的限制又有什么后果？
```

### todo list
java1234 网下载lucene分析与应用 

### 参考资料
+ [https://www.w3cschool.cn/java/javafx-listview.html](https://www.w3cschool.cn/java/javafx-listview.html)
+ [http://www.javafxchina.net/main/](http://www.javafxchina.net/main/)
+ [https://github.com/jfoenixadmin/JFoenix.git](https://github.com/jfoenixadmin/JFoenix.git)
+ [https://github.com/javafxchina/xmdp](https://github.com/javafxchina/xmdp)