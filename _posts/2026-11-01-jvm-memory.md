# JVM （初窥门径）

## 一、JVM 运行时数据区结构
JVM 的运行时数据区主要分为：
- **线程共享区**：Method Area（方法区）、Heap（堆）、执行引擎、本地接口。
- **线程私有（隔离）区**：VM Stack（虚拟机栈）、Native Method Stack（本地方法栈）、Program Counter Register（程序计数器）。
<img width="518" height="420" alt="JVM内存结构" src="https://github.com/user-attachments/assets/c8ed3bcc-f28d-4dd4-8bfa-44f390b7b4cc" />

---

## 二、 源代码字节码深度分析

```java
public class Main {
    public static void main(String[] args) {
        int r = add(2, 3);
        System.out.println(r);
    }

    static int add(int a, int b) {
        int c = a + b;
        return c;
    }
}
```
这是一个简单的 Java 代码，流程是计算 3 + 2 = 5 并输出。这里使用 jclasslib 进行静态字节码分析。

### 1. main 方法字节码分析
```java
public static void main(java.lang.String[]);
    // 0: 将 int 常量 2 压入 VM Stack 的操作数栈。 (Operand Stack: [2])
    0: iconst_2  

    // 1: 将 int 常量 3 压入操作数栈。 (Operand Stack: [2, 3] ，3在栈顶)
    1: iconst_3  

    /* 
     * 2: 调用静态方法。
     * invokestatic: 调用静态方法。 
     * #7: 运行时常量池（Runtime Constant Pool）的索引。JVM 拿着 #7 去常量池查字典，
     *     发现是个 Methodref，进而解析出方法在 Method Area 里的真正入口地址（符号引用 -> 直接引用）。
     * 
     * 【核心动作：跨栈帧传参】
     * 1. 从 main 的操作数栈弹出 (Pop) 3 和 2。
     * 2. 在 VM Stack 中创建 add() 的新栈帧。
     * 3. 将参数压入 add() 的局部变量表：Local[0]=2 (a), Local[1]=3 (b)。
     *    此时 add() 的操作数栈是空的，参数在局部变量表中。
     */
    2: invokestatic #7 <Main.add : (II)I> 

    // 5: add 方法返回 5，将其弹出并存（store）入 main 的局部变量表 Slot 1 (变量 r)。
    5: istore_1  

    // 6: 获取 System.out 静态字段的引用（真正的对象实体在堆 Heap 中），将引用压入操作数栈。
    // getstatic #13 <java/lang/System.out : Ljava/io/PrintStream;> 
    // 1. Resolve static field 'out' from class 'System' (metadata is in Method Area).
    // 2. Get the reference (memory address) pointing to the PrintStream instance.
    // 3. The actual PrintStream object (body) resides in the Heap.
    // 4. Push this reference onto the current Operand Stack.
    6: getstatic #13 <java/lang/System.out : Ljava/io/PrintStream;>  

    // 9: 将局部变量表 Slot 1 (变量 r=5) 的值压入操作数栈。 (Operand Stack: [System.out引用, 5])
    9: iload_1  

    // 10: 调用实例方法 println，打印结果。弹出栈顶的 5 和下方的 System.out 引用。
    10: invokevirtual #19 <java/io/PrintStream.println : (I)V>  

    // 13: main 方法结束，返回 void。
    13: return  
```
### 2. add方法字节码分析
```java
static int add(int, int);
    // 0: 将局部变量表 Slot 0 (参数 a=2) 的值压入操作数栈。
    0: iload_0  

    // 1: 将局部变量表 Slot 1 (参数 b=3) 的值压入操作数栈。 (Operand Stack: [2, 3])
    1: iload_1  

    /*
     * 2: 执行 int 加法。
     * 【逻辑层 (JVM)】: 弹出栈顶 3(右操作数) 和次栈顶 2(左操作数) -> 计算 2+3=5 -> 将 5 压回栈顶。 (Operand Stack: [5])
     * 【物理层 (CPU)】: 解释执行时，JVM(C++)从栈内存取数据放入 CPU 寄存器计算；
     *                  JIT 编译后，直接生成汇编指令 (如 add eax, ebx)，在 CPU 的 ALU 中完成计算。
     */
    2: iadd  

    // 3: 将栈顶的 5 弹出，存入局部变量表 Slot 2 (变量 c)。
    3: istore_2 

    // 4: 将局部变量表 Slot 2 (变量 c=5) 的值压入操作数栈，准备返回。
    4: iload_2  

    // 5: 返回栈顶的 int 值(5) 给 main 方法，同时销毁 add() 栈帧，将 5 压入 main 的操作数栈。
    5: ireturn  
```




