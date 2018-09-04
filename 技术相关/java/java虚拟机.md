### 零散知识点


### 遗留问题
+ 栈映射帧

```
用于加快Java虚拟机中类验证的速度。栈映射帧给出一个方法的执行帧在执行过程中某一时刻的状态。
更准确地说，它给出了在就要执行某一特定字节代码指令之前，
每个局部变量槽和每个操作 数栈槽中包含的值的类型。
```
  + 问题
      + 栈映射信息存放在哪里？可以怎样进行操作

### 相关概念
+ 帧
    + 局部变量 `包含可根据索引 以随机顺序访问的变量`
    + 操作数栈 `包含了供字节代码指令 用作操作数的值`
    
`执行栈中的每一帧都包含自己的操作数栈`

### 指令以及参数
+ `所有其他字节代码指令都仅对操作数栈有效`
+ ILOAD, LLOAD, FLOAD, DLOAD 和 ALOAD 指令读取一个局部变量，并将它的值压到操 作数栈中。它们的参数是必须读取的局部变量的索引 i
+ ILOAD 用于加载一个 boolean、byte、 char、short 或 int 局部变量
+ LLOAD、FLOAD 和 DLOAD 分别用于加载 long、float 或 double 值。(LLOAD 和 DLOAD 实际加载两个槽 i 和 i+1)
+ ALOAD 用于加载任意非基元值，即对 象和数组引用
+ ISTORE、LSTORE、FSTORE、DSTORE 和 ASTORE 指令从操作数栈 中弹出一个值，并将它存储在由其索引 i 指定的局部变量中
+ 栈 这些指令用于处理栈上的值:POP弹出栈顶部的值，DUP压入顶部栈值的一个副本， SWAP 弹出两个值，并按逆序压入它们，等等。
+ 常量 这些指令在操作数栈压入一个常量值:ACONST_NULL压入null，ICONST_0压入 int 值 0，FCONST_0 压入 0f，DCONST_0 压入 0d，BIPUSH b 压入字节值 b，SIPUSH s 压入 short 值 s，LDC cst 压入任意 int、float、long、double、String 或 class1 常量 cst，等等。
+ 算术与逻辑 这些指令从操作数栈弹出数值，合并它们，并将结果压入栈中。它们没有任何 参数。xADD、xSUB、xMUL、xDIV 和 xREM 对应于+、-、*、/和%运算，其中 x 为 I、 L、F 或 D 之一。类似地，还有其他对应于<<、>>、>>>、|、&和^运算的指令，用于 处理int和long值。
+ 类型变换 这些指令从栈中弹出一个值，将其转换为另一类型，并将结果压入栈中。它们对 应于 Java 中的类型转换表达式。I2F, F2D, L2D 等将数值由一种数值类型转换为另一种 类型。CHECKCAST t 将一个引用值转换为类型 t。
+ 对象 这些指令用于创建对象、锁定它们、检测它们的类型，等等。例如，NEWtype指令将 一个 type 类型的新对象压入栈中(其中 type 是一个内部名)。
+ 字段 这些指令读或写一个字段的值。GETFIELD owner name desc 弹出一个对象引用，并 压和其 name 字段中的值。PUTFIELD owner name desc 弹出一个值和一个对象引用，并 将这个值存储在它的 name 字段中。在这两种情况下，该对象都必须是 owner 类型，它 的字段必须为 desc 类型。GETSTATIC 和 PUTSTATIC 是类似指令，但用于静态字段。
+ 方法 这些指令调用一个方法或一个构造器。它们弹出值的个数等于其方法参数个数加 1 (用于目标对象)，并压回方法调用的结果。INVOKEVIRTUAL owner name desc 调用在 类 owner 中定义的 name 方法，其方法􏰂述符为 desc。INVOKESTATIC 用于静态方法， INVOKESPECIAL 用于私有方法和构造器，INVOKEINTERFACE 用于接口中定义的方 法。最后，对于 Java 7 中的类，INVOKEDYNAMIC 用于新动态方法调用机制。
+ 数组 这些指令用于读写数组中的值。xALOAD指令弹出一个索引和一个数组，并压入此索 引处数组元素的值。xASTORE 指令弹出一个值、一个索引和一个数组，并将这个值存 储在该数组的这一索引处。这里的 x 可以是 I、L、F、D 或 A，还可以是 B、C 或 S。
+ 跳转 这些指令无条件地或者在某一条件为真时跳转到一条任意指令。它们用于编译if、 for、do、while、break 和 continue 指令。例如，IFEQ label 从栈中弹出一个 int 值，如果这个值为 0，则跳转到由这个 label 指定的指令处(否则，正常执行下一 条指令)。还有许多其他跳转指令，比如 IFNE 或 IFGE。最后，TABLESWITCH 和LOOKUPSWITCH 对应于 switch Java 指令。
+ 返回 最后，xRETURN和RETURN指令用于终止一个方法的执行，并将其结果返回给调用 者。RETURN 用于返回 void 的方法，xRETURN 用于其他方法。