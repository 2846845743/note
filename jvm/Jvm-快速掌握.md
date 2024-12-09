# Jvm-快速掌握

## 前言

1.什么是JVM？

全称Java 虚拟机，是运行java的必备条件。java虚拟机通过解释字节码指令调用操作系统函数，实现各种功能

由于面向操作系统，所以java是跨平台的，不同系统由不同版本的jvm运行，可以做到一次编译，处处运行。

2.jre：java运行的最小环境，由jvm加上一些核心类库组成

3.jdk：java开发最小环境，由jre加上一些开发工具组成。





![img](https://cdn.nlark.com/yuque/0/2024/png/38443033/1715259492421-1c009573-2988-4e9e-8fa4-fd32f34e83f3.png)

### 运行时数据区

### 1.程序计数器

​	主要用于1.对程序进行顺序控制2.不同线程切换是记录当前线程上次执行到哪里了

### 

### 2.栈

​	记录每个线程运行时独有的一些变量。每执行一个方法就会将该方法压栈，每一个方法称为一个栈帧

每一个栈帧由以下组成：

​	**1.局部变量表：记录着局部变量的引用**

​	**2.操作数栈**

​	**3.动态链接：用于将字节码的符号引用转换成直接引用**： 主要服务**一个方法需要调用其他方法的场景**。Class 文件的**常量池里保存有大量的符号引用**比如方法引用的符号引用。当一个方法要调用其他方法，需要将常量池中指向方法的符号引用转化为其在内存地址中的直接引用。动态链接的作用就是为了**将符号引用转换为调用方法的直接引用**，这个过程也被称为 **动态连接** 。

是不是栈帧越大越好？不是，栈一定的情况下，栈帧越大，可以创建的线程数就越小。

局部变量是不是一定线程安全？不一定，尤其是传参或者返回这个局部变量的时候，变量的具体对象的作用域扩张到方法外了

------



### 3.堆区

java中最大的区域**，1.8之前的堆区包含永久代，永久代是堆的逻辑区域，1.8之后永久代移除到了元空间，并且使用的是本地内存**。任何new出来的对象都会在堆区创建，又分为新生代和老年代，新生代有Eden，from，to=8：1：1组成。方便垃圾的分代回收。

**1.6及以前字符串常量池保存在方法区（永久代）的运行时常量池的字符串常量池中，1.7之后字符串常量池和静态变量移动到了堆区**

**StringTable-字符串常量池**，1.6的时候在运行时常量池，后续移动到了堆区。

字符串常量池主要存放字符串常量信息，new String("ab")这句话创建了两个对象，一个是“ab”字符串对象，放进了串池，另一个是堆区中的String对象。

**intern（）方法：手动将为被放入串池的字符串加入到串池当中。1.8中存放对象的本体，1.6中会拷贝一份再放入。**



### 4.方法区

方法区存放：**类信息、字段信息、方法信息、常量、静态变量、即时编译器编译后的代码缓存等数据**。是一个抽象概念，**永久代和元空间**是方法区的具体实现。

**为什么要将永久代 (PermGen) 替换为元空间 (MetaSpace) 呢?**：因为类加载的越多，方法区占用越大，放在堆区可能不够用，用本地内存就够用了，而且永久代不适合GC，效率很低。

####   运行时常量池：属于方法区的一部分。存放Class常量信息



### 5.直接内存

直接内存不受jvm管控，是一片专门用于NIO操作提高效率的空间，它使得JAVA用户进程可以直接操作内核空间，减少了不必要的数据拷贝（从用户态拷贝到内核），通过一个ByteBuffer来引用这片区域。快速提升io操作。

直接内存通过Unsafe.freeMemery()来释放。**这里还有System.gc（）没看懂，回去看看**





## 垃圾回收

### 1.垃圾的判断

​	1.引用计数法：对象引用了另一个对象，计数器+1.但是解决不了循环引用问

​	2.可达性分析算法：通过一个GcRoot，可以引用的对象。



### 2.五种引用

​	1.强引用：GCRoot对象的直接引用

​	2.软引用：通过SoftReference引用的对象，内存满才会清理。可以配合引用队列使用

​	3.弱引用：通过WeakReference引用的对象，任意一次GC都会清理。可以配合引用队列使用，用来清除这个弱引用对象本身

​	4.虚引用：必须关联引用队列！**虚引用对象会创建一个Cleaner，记录着ByteBuffer对应的直接内存，当ByteBuffer对象被回收掉了，让虚引用对象进入引用队列，然后调用Cleaner的方法记录的直接内存地址，通过Unsfafe释放直接内存。**

​	5.终结器引用：当对象准备被回收（没有强引用了），会将终结器引用加入引用队列，然后通过一个优先级很低的线程调用对象的Object.finallize方法，这样可以在对象被回收之前调用finallize方法。**不建议使用！**



### 3.分代回收机制

堆内存分为新生代和老年代，Eden和from和to比例是8:1:1，new出来的对象首先会放入eden区，eden满之后触发MinorGc会将eden和from的存活对象转移到to中，并且让它的年龄+1，年龄阈值到达15的或者大对象会被加入到老年区。老年区都是一些不容易被回收的对象，所以采用标记清楚或者标记整理算法。



### 4.垃圾回收器

大体分为以下几种

**1.串行回收器：serial new old**

**2.吞吐量优先：并行回收器，parerl new old**

**3.暂停时间优先**：CMS。CMS ：初始标记：并发标记：STW暂停重新标记（很短）：并发清理

当堆内存小的时候，暂停时间相对短，内存大的时候，吞吐量相对好。



**4：G1回收器**

G1回收器是一款综合吞吐量和暂停时间的垃圾回收器，在堆内存很大的时候有着十分优越的优势。

大致流程分为三个阶段：**1.Young Collection新生代回收 2.YoungCollection + concurrent Mark 新生代回收+并发标记 3.Mixed Colleccion混合回收**

**1 young Collection：**堆内存把空间分为大小相同的若干个空间，每一个空间都可以作为eden，from，to ，old。

一开始先随便找一个作为eden并存入，内存不够了就清理并移动到survivor **采用复制算法**，survivor中年龄大的对象会移动到Old区

**2.Young Collection +CM：**在年轻代回收STW的时候会顺便进行初始标记找到GCRoot根对象，当堆中Old到达45%默认阈值，就会触发并发标记（不赞停）。

**3.Mixed GC**：分为两个阶段，最终标记和拷贝清理，都会进行STW，因为上一步并发标记是不暂停的，类似于CMS，所以这一步的STW很短。**为了达到目标的暂停时间，拷贝清理的过程会有选择性的将某几个死亡对象占比多的Old区进行清理，而不是全部，这就是Garbage First的名字由来**



![image-20240822204903268](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240822204903268.png)

CMS和G1的老年代内存不足分为两种情况，如果回收速度比产生垃圾的速度快，那么就称不上是FULLGC，如果产生垃圾的速度更快，来不及回收，那么就称为Full GC。



跨代引用：老年代会分为一个个卡，引用了新生代的卡叫做脏卡，只需单独处理脏卡。

Remark重新标记阶段：已经处理完的对象变成黑色，正在处理的灰色，如果一个对象更改了他的引用，会加入一个写屏障并放入一个队列当中，remark阶段会发生STW，从队列里找并处理。



![image-20240822233425698](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240822233425698.png)

**类的卸载：当一个类加载器所有类都不再使用，类的所有实例都被回收则会卸载这个类**

![image-20240822234152061](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240822234152061.png)



### GC调优

选择合适的回收器，确认你需要吞吐量还是低延迟

![image-20240823000734821](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240823000734821.png)

新生代是不是越大越好？不是，如果新生代开太大，老年代空间就少，老年代少容易发生FullGC导致耗时更长，建议YoungGen在堆的0.25-0.5左右。要寻找一个折中最优点

新生代最好能容量所有【并发量*（请求到相应）】的对象

幸存区大到能保留当前活跃对象+需要晋升对象

晋升阈值配置得当，让存活时间长的对象快点晋升

**老年代调优：**

老年代内存越大越好；先尝试不做调优；观察FullGc老年代内存占用情况，将老年代内存预设调大1/4

---

## 类文件结构-.class

前四个字节为魔数，代表class文件，后面78是标注jdk版本号52代表jdk8.

包含魔数，jdk版本号，常量池，字段信息，方法信息。

```java
Classfile /C:/Users/86183/JAVA/code/API/out/production/API/Web/Javap_Demo.class
  Last modified 2024年8月23日; size 597 bytes
  SHA-256 checksum edaf95ead4c5fb56e9202529c68d3bd990f20fbe398a3fee46101898d7a04854
  Compiled from "Javap_Demo.java"
public class Web.Javap_Demo
  minor version: 0
  major version: 62
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #10                         // Web/Javap_Demo
  super_class: #2                         // java/lang/Object
  interfaces: 0, fields: 1, methods: 2, attributes: 1
Constant pool:	//常量池
   #1 = Methodref          #2.#3          // java/lang/Object."<init>":()V
   #2 = Class              #4             // java/lang/Object
   #3 = NameAndType        #5:#6          // "<init>":()V
   #4 = Utf8               java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = String             #8             // a
   #8 = Utf8               a
   #9 = Fieldref           #10.#11        // Web/Javap_Demo.a:Ljava/lang/String;
  #10 = Class              #12            // Web/Javap_Demo
  #11 = NameAndType        #8:#13         // a:Ljava/lang/String;
  #12 = Utf8               Web/Javap_Demo
  #13 = Utf8               Ljava/lang/String;
  #14 = Fieldref           #15.#16        // java/lang/System.out:Ljava/io/PrintStream;
  #15 = Class              #17            // java/lang/System
  #16 = NameAndType        #18:#19        // out:Ljava/io/PrintStream;
  #17 = Utf8               java/lang/System
  #18 = Utf8               out
  #19 = Utf8               Ljava/io/PrintStream;
  #20 = String             #21            // Hello world
  #21 = Utf8               Hello world
  #22 = Methodref          #23.#24        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #23 = Class              #25            // java/io/PrintStream
  #24 = NameAndType        #26:#27        // println:(Ljava/lang/String;)V
  #25 = Utf8               java/io/PrintStream
  #26 = Utf8               println
  #27 = Utf8               (Ljava/lang/String;)V
  #28 = Utf8               Code
  #29 = Utf8               LineNumberTable
  #30 = Utf8               LocalVariableTable
  #31 = Utf8               this
  #32 = Utf8               LWeb/Javap_Demo;
  #33 = Utf8               main
  #34 = Utf8               ([Ljava/lang/String;)V
  #35 = Utf8               args
  #36 = Utf8               [Ljava/lang/String;
  #37 = Utf8               SourceFile
  #38 = Utf8               Javap_Demo.java
{
  public Web.Javap_Demo();				//方法信息
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: ldc           #7                  // String a
         7: putfield      #9                  // Field a:Ljava/lang/String;
        10: return
      LineNumberTable:
        line 3: 0
        line 5: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      11     0  this   LWeb/Javap_Demo;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #14                 // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #20                 // String Hello world
         5: invokevirtual #22                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 8: 0
        line 9: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
}

```



**一个main方法的执行流程：先编译，加载这个类，将常量池加载到运行时常量池，将类的信息加载到方法区。然后将当前方法加入栈帧，创建局部变量表，然后执行对应字节码指令，最后返回**。如果常量池数字大于32665就会存到运行时常量池，否则会跟字段存在一起。



### Init方法

1.cinit：类的初始化方法，包括静态变量和静态代码块赋值，jvm会将static变量，static{}代码块整合成一个新的方法，按顺序执行

2.init：实例的初始化方法，包括构造方法。按顺序将{}代码块，和构造方法（构造方法会放最后执行）合并成一个方法，执行。



### 多态原理

![image-20240824124744698](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240824124744698.png)





### Try，catch ,finally原理。



以一种情况：对于try,catch,finlly的情况，字节码层面会分成三个分支。**1.try代码块 2.catch捕获成功代码块 3.catch捕获不到的代码块****，字节码指令会在这三个代码块的后面都复制一份finally的内容。所以无论如何finally都会执行。

第二种情况：只有try和finally：这时候只有两个分支，也是会在每个分支后面复制一块finally代码块字节码执行。**注意：最好不要在finally中return，这样可能会导致无法抛出异常，因为字节码指令athrow在没捕获异常但是发生异常的时候会在最后才执行，如果在finally就return了可能导致无法抛出**



在try中return之后，会先将这个变量暂存到操作数栈栈顶，然后执行finally的语句，最后return这个栈顶数据。如果finally也有return，同样暂存到栈顶，最后返回的是finally中的栈顶数据





### 语法糖：简单了解了一下，过程省略。



### 类加载

1.加载：

![image-20240828215735911](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240828215735911.png)

把类class加载到元空间

2.**链接**：分为三步

​	验证：验证class文件是否符合jvm规范

​	准备：为类的静态变量分配空间。注意，类静态变量1.8以后是在堆区。final修饰的基本常量或者字符串常量会在这个时候顺便赋值了。其他会在初始化的时候赋值

​	解析:将常量池中的符号引用解析为直接引用

3.**初始化**：

类的初始化，即cinit，类的构造方法。static静态代码块从上到下会组成一个cinit方法。有几种操作会直接触发初始化：1.main的类  2.访问类的静态变量 3.new一个类 4.class.forname。5.子类初始化带动父类初始化 

![image-20240829010936938](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240829010936938.png)

以上代码谁会触发初始化？：只有访问c的时候触发。



 

```java

/**
 * 单例：懒汉加载
 */
public class SingleTon {
    public static void main(String[] args) {
        SingleTon.getInstance();
    }

    static {
        System.out.println("SingelTon类的初始化");
    }
    public static   void test(){

    }
    private SingleTon(){}

    public static class LazyHolder{
        //静态内部类
        private static final SingleTon SINGLE_TON = new SingleTon();

        static {
            System.out.println("Lazy的加载");
        }


    }
    public  static SingleTon getInstance(){
        return LazyHolder.SINGLE_TON;
    }
}

```

单例模式的简单经典实现。只有调用getInstance方法的时候才会走LazyHolder的初始化，而且这个对象只有一份。





### 类加载器

**启动类加载器：/jre/lib**

**扩展类加载器：/jre/lib/ect**

**应用程序加载器：classPath**

加载一个类时，会先访问parent，如果父加载器没有加载过，才会调用自己的加载





JVM运行期优化：1.逃逸分析：对于热点代码会进行缓存，不用每一次都解释。而且如果发

发现循环内创建对象但是没有引用，会直接不创建对象

2.方法内联：会将简单的方法直接嵌入写到调用者那里

3.字段优化：
