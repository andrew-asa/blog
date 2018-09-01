### 零散知识点
+ 窗口居中 先show再进行调用

```
private void setWindowCenter(Stage stage) {

        Rectangle2D primaryScreenBounds = Screen.getPrimary().getVisualBounds();
        stage.setX(primaryScreenBounds.getMinX() + (primaryScreenBounds.getWidth() - stage.getWidth()) / 2.0);
        stage.setY(primaryScreenBounds.getMinY() + (primaryScreenBounds.getHeight() - stage.getHeight()) / 2.0);
    }
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

+ 字体
    + 创建字体
    ```       
     Font font =  Font.font(family, size); 
    ```
+ JavaFX Scene Builder

``UI设置与代码分离，等同于chrome的控制台？``

todo list
java1234 网下载lucene分析与应用 
