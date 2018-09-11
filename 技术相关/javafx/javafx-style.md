### 零散知识点
+ scene 中添加样式

    ```
    scene.getStylesheets().add("ControlStyle.css")
    ```

   + 样式写法
   
   ```
   .button1{
    -fx-font: 22 arial;
    -fx-base: #b6e7c9;    
   }
   ```

+ node 中添加类

```
button1.getStyleClass().add("button1")
```

### Q
+ 可以看到其实和html中添加样式一样,但是为什么要加上-fx-前缀呢？本来是可以共用的东西。
+ .setStyle("-fx-font-size: 15pt;")为什么不可以多次添加合并？
+ 添加类的时候是否会进行刷新界面？还是仅仅是局部刷新而已？