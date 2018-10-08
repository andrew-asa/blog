### 零散笔记
+ AffineTransform 仿射变换
+ Graphics2D 相关
+ 绘画出来的字体模糊
+ setComposite
+ BasicStroke
+ AttributedString
+ 像素密度
+ GraphicsEnvironment
+ BufferedImage 
+ 逻辑像素，物理像素
+ FontRenderContext
### Graphics2D
+ setRenderingHints 设置绘图提示，它提供了速度与绘图质量之间的一种平衡
+ setStroke 设置笔划，笔划用于绘制形状的边框。可以选择边框的粗细和线段的虚实。
+ setPaint 方法来设置着色方法。着色方法用于填充诸如笔划路径或者形状内部等区域的颜色。可以创建单色的着色法，也可以用变化的色调进行着色，或者使用平铺的填充模式。
+ clip方法来设置剪切区域。
+ transform方法，设置一个从用户空间到设备空间的变换方式。如果使用变换方式比使用像素坐标更容易地定义在定制坐标系统中的形状，那么就可以使用变换方式。以用变化的色调进行着色，或者使用平铺的填充模式。
+ deriveFont 可以放大字体
   
   ```        
BufferedImage image = new BufferedImage(imgWidth ,imgHeight,BufferedImage.TYPE_INT_RGB);
```
+ 获取像素数据 

```
DataBufferInt dataBuffer = ((DataBufferInt) image.getRaster()
              .getDataBuffer());
```
  
+ bufferimage 生成的graphics 2d和awt控件绘画的graphics2d 是同一个对象都是SunGraphics2D
+ AffineTransform[[2.0, 0.0, 0.0], [0.0, 2.0, 0.0]]

### RenderingHints 参数
+ KEY_RENDERING 确定着色技术，在速度和质量之间进行权衡。可能的值有 VALUE_RENDERING_SPEED, _QUALITY 或 _DEFAULT。
+  renderHint = SunHints.INTVAL_RENDER_DEFAULT;
+  antialiasHint = SunHints.INTVAL_ANTIALIAS_OFF;
+  textAntialiasHint = SunHints.INTVAL_TEXT_ANTIALIAS_DEFAULT;
+ textAntialiasHint = SunHints.INTVAL_TEXT_ANTIALIAS_ON;
+ fractionalMetricsHint = SunHints.INTVAL_FRACTIONALMETRICS_OFF;
+ lcdTextContrast = lcdTextContrastDefaultValue;
         interpolationHint = -1;
         
### 可以操作项
+ 使用抗锯齿处理和微调（hinting）以达到更好的输出质量
+ 可以使用系统安装的所有字体
+ 可以将对图形对象的操作（旋转、缩放、着色、剪切等等）应用到文本上。
+ 支持向字符串添加内嵌属性（如字体、尺寸、深浅，甚至图像）
+ 支持双向文本（启用从右到左的字符顺序，就象您在阿拉伯语和希伯来语中可能遇到的一样）
+ 第一光标和第二光标能够浏览同时包含从右到左和从左到右字符顺序的文本。
+ 先进的字体度量功能，超过旧的 java.awt.FontMetrics 类中的相应功能
+ 排版功能可以实现单词换行和调整多行文本

### 参考资料
+ 1:[https://blog.csdn.net/cping1982/article/details/5376406](https://blog.csdn.net/cping1982/article/details/5376406)
+ 2:[awt api中文文档](https://tool.oschina.net/uploads/apidocs/jdk-zh/index.html?java/awt/RenderingHints.html)
+ 3:[https://blog.csdn.net/fykhlp/article/details/6204714](https://blog.csdn.net/fykhlp/article/details/6204714)
+ 4:[https://doc.yonyoucloud.com/doc/jdk6-api-zh/java/awt/Graphics2D.html](https://doc.yonyoucloud.com/doc/jdk6-api-zh/java/awt/Graphics2D.html)
+ 5:[http://iwebcode.iteye.com/blog/1791878](http://iwebcode.iteye.com/blog/1791878)
+ 6:[https://stackoverrun.com/cn/q/3621640](https://stackoverrun.com/cn/q/3621640)