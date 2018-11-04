### 知识点
+ Parent
    + lookup(selector) 像前端那样提供选择器查找子类
    + requestLayout 请求父节点对其拥有的子节点进行布局
+ Pane
    + 节点居上和居左对齐方式。放在其上面的元素需要制定x，y坐标,体现为绝对布局
    + Pane 添加child的时候回都放到左上角，所以需要Pane.xxx相应的方法设置child node的位置
   
+ VBox `单列布局`
   + setPadding 设置子节点之间的间隔
    
+ HBox
   + 节点是居上和居左对齐。
   + 区别于Pane是可以在x方向进行自增的。
   
+ Priority 设置子子节点自适应的属性
+ Group
+ TilePane
   + 每个单元格(Tile)的大小一致。每块Tile的大小由所有按钮的最大优先宽度和高度决定。
   + 当窗口大小变化时，Tile的大小不会跟着变化，因此其中的按钮大小也不会变化。注意一旦窗口被缩小了，在TilePane中的按钮会改变位置，但并不会随之缩小。
   + 节点会居中对齐

+ FlowPane
   + 节点会居中对齐

+ BorderPane
   + 上、下、左、右、中

+ StackPane
    + 添加组件
```
stackPane.getChildren().add(imageView);
stackPane.getChildren().add(text);
```
    
    + 设置组件中的绝对布局
    
```
Insets insets = new Insets(0, 30, 30, 0);
StackPane.setAlignment(text, Pos.BOTTOM_RIGHT);
StackPane.setMargin(text, insets);
```
先添加进去的组件会放在下面
   
+ 布局主要考虑几个问题
    + 节点的大小变化
    + 节点的位置
    + 每一个布局管理器都可能有其特性，主要体现在当窗口变大变小时会如何改变里面节点的大小以及位置


+ 布局通用属性
   + setAlignment()方法来控制节点和面板的对齐方式。


   
+ ListView `listview能够解决动态添加元素的机制`
    + 禁止滚动条
    + 动态高度
    
# Q
+ 自增长的pane是否存在？
+ Pane存放child是叠放在一起的这是由什么属性决定的？
+ Pane 增大的时候 scence和stage并不会增大，应该如何进行重新绘制？

# 参考资料
+ 1:[https://www.cnblogs.com/yangwen0228/p/6831486.html](https://www.cnblogs.com/yangwen0228/p/6831486.html)
+ 2:[https://stackoverflow.com/questions/14983706/javafx-stackpane-x-y-coordinates](https://stackoverflow.com/questions/14983706/javafx-stackpane-x-y-coordinates)
+ 3:[JavaFX布局（一） https://blog.csdn.net/theonegis/article/details/50184811](https://blog.csdn.net/theonegis/article/details/50184811)
+ 4:[HBox VBox 布局利用Priority实现布局自适应](https://blog.csdn.net/cdc_csdn/article/details/80710001)
      
	