### 零散知识点
+ 默认程序打开文件`desktop.open(file);`
+ 窗口居中 先show再进行调用
+ canvas转图片还需要启动一个javafx线程

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
    + stage 可以想成是視窗，基本上class Stage 也是繼承視窗class Window。所以有關視窗視窗的設定就要先找到這個物件來操作。
   + scene 可以當做是把視窗的外框去掉後，中間那一塊區域。
  + node 則是區域中的所有元素。由一個 rootnode 當作樹狀結構的根節點開始展開。
從「scene.getWindow()」可以找到「stage」
從「scene.getRoot」可以找到「rootnode」
    + 设置应用程序图标 `stage.getIcons().add(new Image("file:xxx.png"));`

+ Scene
    + 设置背景透明 `setFill(null)`



+ JavaFX Scene Builder
    + FXMLLoader.load `反序列化，在Scene Builder里面先完成控件，界面的布局设置，然后调用loader出来进行反序列化成对应的布局，控件`

``UI设置与代码分离，等同于chrome的控制台？``

### 观点
+ 一个jvm只能运行一个javafx程序
+ 如果像label只是整个javafx的一个节点，那么在整棵树中应该怎样进行增删改查，以及界面是什么时候，用一种什么样的方式进行刷新。似乎刷新机制才是整个javafx的重点。
+ 为什么ui需要在不同的线程中进行刷新？

```
为什么，是什么做了限制？
```

+ 创建和关闭Stage只能通过javafx提供的线程里面进行创建

```
为什么，为什么要做这样的限制，如果不进行这样的限制又有什么后果？
```

### todo list
+ java1234 网下载lucene分析与应用 
+ 搜索输入文字下拉展开
  + 用listview 但是却不能解决listview中如何不显示问题以及listview 中高度随子节点高度动态调整的问题。
  + 用ComboBox里面却不能放node节点元素，所以赶紧要解决这个问题还得深入了解弹出框，下拉框，ComboBox等底层原理。

### 参考资料
+ [https://www.w3cschool.cn/java/javafx-listview.html](https://www.w3cschool.cn/java/javafx-listview.html)
+ [http://www.javafxchina.net/main/](http://www.javafxchina.net/main/)
+ [https://github.com/jfoenixadmin/JFoenix.git](https://github.com/jfoenixadmin/JFoenix.git)
+ [https://github.com/javafxchina/xmdp](https://github.com/javafxchina/xmdp)
+ [在线api https://docs.oracle.com/javase/8/javafx/api/toc.htm](https://docs.oracle.com/javase/8/javafx/api/toc.htm)