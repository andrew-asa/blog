### 零散只是点
+ 窗口居中 先show再进行调用

```
private void setWindowCenter(Stage stage) {

        Rectangle2D primaryScreenBounds = Screen.getPrimary().getVisualBounds();
        stage.setX(primaryScreenBounds.getMinX() + (primaryScreenBounds.getWidth() - stage.getWidth()) / 2.0);
        stage.setY(primaryScreenBounds.getMinY() + (primaryScreenBounds.getHeight() - stage.getHeight()) / 2.0);
    }
```

todo list
java1234 网下载lucene分析与应用 
