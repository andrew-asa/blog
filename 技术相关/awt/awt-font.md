# 零散笔记
+ 获取当前系统支持的字体

```
GraphicsEnvironment e = GraphicsEnvironment.getLocalGraphicsEnvironment();
String[] fontName = e.getAvailableFontFamilyNames();
for(int i = 0; i<fontName.length ; i++) {
    System.out.println(fontName[i]);
}
```

# Q
+ Font.loadFont(this.getClass().getClassLoader().getResourceAsStream("iconfont.ttf"), 12) 没有指定family，但是给出的font里面就已经有，所以猜想是ttf文件里面本身就有字体family的信息

# 参考资料
+ 1:[https://www.zhihu.com/question/20694829](https://www.zhihu.com/question/20694829)