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
	
	
	
+ JavaFX Scene Builder
    + FXMLLoader.load `反序列化，在Scene Builder里面先完成控件，界面的布局设置，然后调用loader出来进行反序列化成对应的布局，控件`

``UI设置与代码分离，等同于chrome的控制台？``

todo list
java1234 网下载lucene分析与应用 
