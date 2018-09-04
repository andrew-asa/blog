### 零散知识点

+ 添加方法以及变量入栈
      + 调用"+"运算
+ 类描述符 见（ASM使用指南）

```
// 添加add方法
	mv = visitor.visitMethod(ACC_PUBLIC, "add", "(II)I", null, null);
	mv.visitCode();
	mv.visitVarInsn(ILOAD, 1);
	mv.visitVarInsn(ILOAD, 2);
	mv.visitInsn(IADD);
	mv.visitInsn(IRETURN);
	mv.visitMaxs(2, 3);
	mv.visitEnd();

```

### 工具处理
+ 查看字节码

```
javap -c xxx.class
```

+ 定义类以及使用、根据bytes生成类

```
Class<?> clazz = classLoader.defineClass("asm.demo.AddOperImpl", cw.toByteArray());
Method addMethod = clazz.getMethod("add", int.class, int.class);
Object result = addMethod.invoke(clazz.newInstance(), 10, 20);

```

### 相关变量解释
+ V1_5 类的版本--java 1.5
+ ACC_XXX 与Java修饰符对应标志 如ACC_PUBLIC
+ ASM4 


### 相关类/方法解释
+ ClassReader
+ ClassWriter

+ ClassVisitor
	  + 使用
	     + 构造 `ClassVisitor cv = new ClassVisitor(ASM4, cw) { }`
	     + ClassWriter 本身就是一个ClassVisitor，可以说是最外层的visitor？
	  + 删
	  
	  ```
	  除了在转发的方法调用中使用经过修改的参数之外，还可以选择根 本不转发该调用。其效果就是相应的类元素被移除。
	  ```
	  
     + visitMethod
     + visitOuterClass
     + visitInnerClass
     + public void visit(int version, int access, String name,
String signature, String superName, String[] interfaces)  
          + version版本号  
    
+ MethodVisitor
     + AnnotationVisitor visitAnnotationDefault();
     + AnnotationVisitor visitAnnotation(String desc, boolean visible);
     + AnnotationVisitor visitParameterAnnotation(int parameter,String desc, boolean visible);
     + void visitAttribute(Attribute attr);
     + void visitCode(); `开始访问代码`
     + void visitFrame(int type, int nLocal, Object[] local, int nStack,Object[] stack);
     + void visitInsn(int opcode);
     + void visitIntInsn(int opcode, int operand);
     + void visitVarInsn(int opcode, int var);
     + void visitTypeInsn(int opcode, String desc);
     + void visitFieldInsn(int opc, String owner, String name, String desc);
     + void visitMethodInsn(int opc, String owner, String name, String desc);
     + void visitInvokeDynamicInsn(String name, String desc, Handle bsm,Object... bsmArgs);
     + void visitJumpInsn(int opcode, Label label);
     + void visitLabel(Label label);
     + void visitLdcInsn(Object cst);
     + void visitIincInsn(int var, int increment);
     + void visitTableSwitchInsn(int min, int max, Label dflt, Label[] labels);
     + void visitLookupSwitchInsn(Label dflt, int[] keys, Label[] labels);
     + void visitMultiANewArrayInsn(String desc, int dims);
void visitTryCatchBlock(Label start, Label end, Label handler,String type);
     + void visitLocalVariable(String name, String desc, String signature,Label start, Label end, int index);
     + void visitLineNumber(int line, Label start);
     + void visitMaxs(int maxStack, int maxLocals); `设置方法的栈和本地变量表的大小`
     + void visitEnd();
     + 调用顺序
     
     ```
 visitAnnotationDefault?
( visitAnnotation | visitParameterAnnotation | visitAttribute )*
( visitCode
( visitTryCatchBlock | visitLabel | visitFrame | visitXxxInsn |
visitLocalVariable | visitLineNumber )*
visitMaxs )?
visitEnd
     ```
 
+ Label `代表跳转的字节码位置`
+ TraceClassVisitor
+ Type
    + 根据类获取内部名字 `new ClassReader(Type.getInternalName(Person.class))`
    + getDescriptor 返回描述符 例如 
    + getArgumentTypes
    + getReturnType
 
### 总结点
+ reader代表类的访问 `事件生产者`
+ reader接受visitor的访问 `事件消费者`
+ visitor联系write `事件过滤者`
+ 所以可以定制visitor从而自己定制reader获取得到的class字节码


### 遗留问题
+ jdk.internal.org.objectweb.asm.ClassVisitor 和 org.objectweb.asm.ClassVisitor
+ Java 虚拟机规范
+ asm进行字节码操作的时候有没有考虑到

### demo 分析
+ get方法生成
    + bean定义
    
 ```
 package pkg;
    public class Bean {
      private int f;
      public int getF() {
       return this.f;
      }
      public void setF(int f) {
       this.f = f;
      } 
}
 ```
   + getF字节码
   
   ```
   ALOAD 0
    GETFIELD pkg/Bean f I
    IRETURN
   ```
   
   + 生成getF字节码
   
   ```
    mv.visitCode();
    mv.visitVarInsn(ALOAD, 0);
    mv.visitFieldInsn(GETFIELD, "pkg/Bean", "f", "I");
    mv.visitInsn(IRETURN);
    mv.visitMaxs(1, 1);
    mv.visitEnd();
   ```
   
   + set字节码
   
   ```
   ALOAD 0
    ILOAD 1
    PUTFIELD pkg/Bean f I
    RETURN
   ```
 
   + checkAndSetF方法
   
   ```
   public void checkAndSetF(int f) {
      if (f >= 0) {
       this.f = f;
      } else {
       throw new IllegalArgumentException();
      }
   }
   ```
	
	+  checkAndSetF方法 字节码
	
	```
	ILOAD 1
IFLT label
ALOAD 0
ILOAD 1
PUTFIELD pkg/Bean f I GOTO end
label:
      NEW java/lang/IllegalArgumentException
      DUP
      INVOKESPECIAL java/lang/IllegalArgumentException <init> ()V
      ATHROW
end:
RETURN
	```
	
	+ checkAndSetF methodVisitor生成流程
	
	```
	mv.visitCode();
    mv.visitVarInsn(ILOAD, 1);
    Label label = new Label();
    mv.visitJumpInsn(IFLT, label);
    mv.visitVarInsn(ALOAD, 0);
    mv.visitVarInsn(ILOAD, 1);
    mv.visitFieldInsn(PUTFIELD, "pkg/Bean", "f", "I");
    Label end = new Label();
    mv.visitJumpInsn(GOTO, end);
    mv.visitLabel(label);
    mv.visitFrame(F_SAME, 0, null, 0, null);
    mv.visitTypeInsn(NEW, "java/lang/IllegalArgumentException");
    mv.visitInsn(DUP);
    mv.visitMethodInsn(INVOKESPECIAL,
    "java/lang/IllegalArgumentException", "<init>", "()V");
    mv.visitInsn(ATHROW);
    mv.visitLabel(end);
    mv.visitFrame(F_SAME, 0, null, 0, null);
    mv.visitInsn(RETURN);
    mv.visitMaxs(2, 2);
    mv.visitEnd();
	```
	
	+ try-catch处理
	
	```
	public static void sleep(long d) {
      try {
       Thread.sleep(d);
      } catch (InterruptedException e) {
       e.printStackTrace();
      }
   }
	```
	+ 字节码
	
	```
	TRYCATCHBLOCK try catch catch java/lang/InterruptedException try:
      LLOAD 0
      INVOKESTATIC java/lang/Thread sleep (J)V
      RETURN
catch:
      INVOKEVIRTUAL java/lang/InterruptedException printStackTrace ()V
      RETURN
	```
		
### 参考资料
+ 1.[http://yunshen0909.iteye.com/blog/2219540](http://yunshen0909.iteye.com/blog/2219540)
+ 2.[https://blog.csdn.net/aesop_wubo/article/details/48948211](https://blog.csdn.net/aesop_wubo/article/details/48948211)
