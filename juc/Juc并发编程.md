# Juc并发编程

### 进程与线程

进程：进程是程序的一个实例，大部分程序可以多开，当一个程序被运行，从磁盘中加载程序代码到内存中执行，这就叫开启了一个进程。

线程：一个进程分为一到多个线程。一个线程就是一个具体的指令流，java中线程就是最小调度单位。

**对比：线程是进程的子集，进程拥有共享的资源如内存空间，进程间通信比较复杂需要经过网络协议。线程通信较为简单，他们共享了进程的空间**；进程上下文切换很复杂，线程上下文切换十分简单。



### 并行与并发

CPU被轮流使用，**同一时间应对多件事情的能力叫做并发**。把CPU时间片分给不同的线程使用，让他们看起来是并行执行的，“微观串行，宏观并行”。

**真正同时运行不同指令的能力就叫并行**，比如多核cpu分别处理多个线程任务。叫做并行。



### 异步调用

从方法调用的角度：如果需要等待方法返回就叫同步，否则就叫异步。



**单核cpu执行多线程并不会提高效率，因为实际上他仍然是串行执行的，多核如果将代码拆分成不同任务，会很大提高程序执行效率，但是线程并不是开越多越好，对于cpu计算密集型开N+1个，io密集型开2N个。**





### Java线程

**创建线程的方式**

1.继承Thread并重写run方法

2.使用Runnable重写接口run方法，将这个任务对象作为参数传给Thread，“组合优于继承”。这种方式将任务和线程两个概念分开，第一种则是没有分开

3.使用FutureTask配合Callable，将Callable任务对象传递给FutureTask，然后FutureTask作为Thread的参数，区别是可以获得线程运行的返回值，并且线程内部可以抛出异常。

**查看进程的方法**

windows下用tasklist和taskkill查看/杀死

linux用ps，或者top

java提供jstack，jps等命令查看。

**常见方法**

1.start vs run

start才是真正开启了线程，run只是在原有的线程中执行了这个run方法。

2.sleep和yield

**调用SLeep会让Running状态变成Time Waiting。其他方法可以用interrupt打断正在睡眠 的线程并且sleep会抛出异常；睡眠结束未必立刻执行；建议使用TimeUnit的Sleep代替**

调用yield只是让RUnning回到Runnable状态，可以说没什么卵用。



### Join

```java
public static void test2() throws InterruptedException {
        Thread t1 = new Thread(()->{
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
        Thread t2 = new Thread(()->{
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
        t1.start();
        t2.start();
        long start = System.currentTimeMillis();
        t1.join();//当前线程放在t1之后执行
        t2.join();
        long end = System.currentTimeMillis();
        System.out.println(end-start);
    }
```

输出多少？答案是两秒。因为等待t1时已经等了一秒，那么t2只需要再等1秒就会结束，线程结束就会继续执行主线程任务。

**join还可以增加等待时间参数，如果超过时间线程还没结束也继续执行自己的任务了**





**打断睡眠中的线程，或者wait，join**之后打断标记会变成假false。

**打断正常运行的线程后打断标记为真**

打断标记的作用是用来优雅的暂停正在运行的线程

一个简单的demo：在while循环中每次判断线程标记，如果为真则break退出表示线程被打断了。



**interrupt还可以让park失效，park可以暂时让线程停止，使用interrupt将打断标记设置为真之后park就失效了，就会继续执行后面的代码。只有打断标记为假的时候park生效。**

过时方法：stop（）  suspend（） resume（）这些过时了不要再用。



**守护线程：其他线程都执行完了，守护线程即使没执行完程序也会停止。人如其名，它只是陪着非守护线程走到最后的线程。例如垃圾回收器线程**





**线程状态：操作系统层面五种**

![image-20240902221613474](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240902221613474.png)





**线程状态：java内定**

new 刚新建但是还没start

runnable 就绪状态，可以获得cpu时间片

waiting 调用了wait或者join的无超时等待

time_waiting sleep的超时等待

blocking 获取锁失败的阻塞

terminal 执行完的线程。





### 线程安全问题



两个线程同时执行非原子的代码，发生上下文切换时会导致没来得及写入主存而读取了旧数据，这就叫线程安全问题。为了解决这一问题，Java提供了Synchronized锁来保护临界区的变量安全。

**synchronized代码块不代表一直霸占了cpu，它任然会放出cpu时间片给别人，只不过因为别人没能获取锁，过一段时间又会换回来自己这里，直到执行完代码把锁释放，别人获得到了对象锁才可以执行它自己的任务。**



1.加在成员方法上，锁住了当前类的this实例对象

2.加在静态方法上，锁住了整个类class对象



**成员变量很容易发生线程安全问题**

对于局部变量，如果局部变量引用暴露给了其他线程，例如。

```java
class ThreadSafe {
 public final void method1(int loopNumber) {
 ArrayList<String> list = new ArrayList<>();
 for (int i = 0; i < loopNumber; i++) {
 method2(list);
 method3(list);
        }
    }
 private void method2(ArrayList<String> list) {
 list.add("1");
北京市昌平区建材城西路金燕龙办公楼一层   电话：400-618-9090
    }
 private void method3(ArrayList<String> list) {
 list.remove(0);
    }
 }
 class ThreadSafeSubClass extends ThreadSafe{
 @Override
 public void method3(ArrayList<String> list) {
 new Thread(() -> {
 list.remove(0);
        }).start();
    }
 }
```

方法三子类重写了方法，并且创建新县城来处理这个list的引用，就会发生线程安全问题。我们可以学习String类，对于不会被继承的类加上final，防止子类重写导致的线程安全问题。



String类，Integer类这种不可变类是没有线程安全问题的，它将引用地址指向另一个对象，而不是改变了这个对象的属性。线程安全集合如HashTable的每一个方法都是原子性的，但是他们的组合不是原子的





**Monitor-synchronized原理**

每一个java对象都会关联一个由c++实现的监视器：monitor。这个监视器会在java对象获取到锁之后存放在对象的对象头中的MarkWord当中，前30位代表monitor地址，后2位为锁的状态。



**锁优化-轻量级锁**

当多个线程对同一个对象加锁，但是他们并没有竞争的情况下，会加轻量级锁提升效率。

1.线程对象创建锁记录，存放着锁记录地址和锁计数器。

2.关联java对象obj。

3.obj的对象头markword的信息会和锁记录地址进行交换，交换过程使用cas保证原子性。

**如果交换失败，说明其他线程已经获取了这个轻量级锁，这时候抢锁失败，会升级成重量锁（锁膨胀）**

4.如果遇到锁竞争会升级成重量级锁。

5.锁重入的时候，线程对象给自己加多一个锁记录，并且让顶部锁记录计数器+1。

6.释放锁的时候，如果锁记录计数器为null，表示有重入锁，则直接把这个锁记录删掉。

7.如果锁记录计数器不为null，则可以退出锁，把对象头信息交换回来

**（如果交换失败，说明刚刚有人跟他抢锁，它升级成了重量锁，则会走重量锁的释放流程）**



**自旋优化：如果一个线程获得了重量锁，另一个线程想要获取锁的时候失败，不会立即进入等待队列，而是先自旋重试几次，如果刚好线程1释放了锁，则可以直接获得这个锁**



**偏向锁：为了解决轻量级锁的重入不断cas的问题，偏向锁的对象头markword结构不再是和锁记录交换，而是保存当前线程的线程id，这洋重入的时候就不用cas 了，看到线程id是自己就直接计数器+1，并把Hashcode信息丢弃，所以调用了HashCode方法就会默认禁用掉偏向锁，因为hashCode没地方存了。轻量级锁和重量级锁都会把HashCode存到别的地方。**

（偏向锁的什么鸡巴撤销，重新偏向没搞懂，需要时回去再看。）

![image-20240903215847930](C:\Users\86183\AppData\Roaming\Typora\typora-user-images\image-20240903215847930.png)





### Wait和notify

已经获取锁的线程obj调用wait的时候，线程会进入waitset等待队列，然后交出锁的使用权和cpu的时间片，待同一把锁的另一个线程notify唤醒之后，才会重新一起抢锁，未必就一定抢到。线程状态会变成waiting或者time_waiting，就看有没有加时间参数了。

**wait和sleep的区别：**wait会让时间片也会让出锁，sleep不会让出锁，wait是obj的方法，sleep是线程的方法。wait需要获得锁才可以用，sleep不需要





设计模式之保护性暂停：

```java
//保护性暂停：类似于join的底层原理，并且增加了超时效果。
class GuardedObject{
    private Object response;

    public GuardedObject(int id) {
        this.id = id;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    private int id;//标识该GuardedObject
    /**
     *
     * @param timeout :表示要等待的最大时间，超过了就直接唤醒
     * @return
     */
    public Object get(long timeout){
        synchronized (this){
            //记录开始等待时间；
            long begin=System.currentTimeMillis();
            //经过的时间：
            long passTime=0;
            while(response==null){
                //waitTime表示此轮循环应该等待的时间
                long waitTime=timeout-passTime;
                //经历的时间超过了最大等待时间，则退出循环
                if(waitTime<=0){
                    break;
                }
                try {
                    this.wait(waitTime);//另一线程虚假在最大等待时间之内唤醒了你，这是response依然是空，下一次循环进来还等timeout秒吗？这样就长了。
                }
                catch (InterruptedException e){
                    e.printStackTrace();
                }
                //求得经历的时间
                passTime = System.currentTimeMillis() - begin;
            }
            return response;
        }
    }

    public void complete(Object response){
        synchronized (this){
            //给结果成员变量赋值
            this.response=response;
            this.notifyAll();
        }
    }

}

```

例如，现在有a，b两个线程在wait，但是主线程想要叫醒b，使用notifyall无意间叫醒了a，但是a需要的东西并没有送过来，就发生了“虚假唤醒”。



**Park和Unpark**

属于一个工具类LockSport，调用park线程将暂停运行，好处是不需要获取锁，其他线程利用unpark唤醒。注意，先调用unpark再调用park也是可以的。这会使线程对象的park实例中count+1，然后调用park的时候会优先查看这个count是1吗？是的话就吃掉它，然后继续执行。不是1才会暂停。（旅人补充干粮例子，多个unpark只会补充一个干粮）





**线程状态转换深入理解：waiting和timeWating，跟锁不锁没关系，跟调用什么api有关系，只需要搞清楚api就会了，有很多种情况，慢慢分析。**



**多把锁，在业务不相关的时候，可以拆分锁的粒度。**



### 死锁：线程A想获取线程B已经获得的锁，线程b又等a释放a锁，则发生死锁。

死锁的检测：利用jps和jstack。2用jconsloe



#### 哲学家就餐问题：5个哲学家，五个筷子，筷子代表锁，如果正常情况下大家轮流吃饭就不会发生死锁，但是cpu运转是很快很随机的，某一时刻五个人都获得了分别5个筷子，然后他们都在等待别人释放筷子，就会造成死锁。



#### 活锁：如果两个线程互相改变对方期望的条件，让对方无法结束线程，就叫活锁。例如count变量，线程1需要100可以结束，对他进行++操作，线程2需要count为0才能结束，但是对他进行--操作。导致线程1无法结束

解决方案：增加随机的睡眠时间，让两个线程的执行时间不同，自然就有一个线程快，更容易结束。





**解决死锁的一种小办法是：按照顺序获得a,b两个锁，线程1按照ab，线程2也按照ab而不是ba，这洋可以避免死锁，但是同时会引发“饥饿”问题，导致线程2一直获取不到锁，执行不到代码。**





### ReentrantLock

**对比synchronized有以下特性：可打断，可公平，可设置超时**

1.reentrantLock是可重入的

2.如果其他线程等锁，等到一半不想等了，可以打断，而sync不行。**调用lock.lockInterruptbly方法**

被打断后会抛出异常，捕获这个异常走其他逻辑，既可实现中断

3.锁超时：可以避免死锁，如果获取锁的线程一直阻塞，超过一定的时间参数，我就不等了。lock.tryLock()方法。获取不到锁返回false

4.默认是非公平锁，因为一般情况下设置公平锁没什么必要。

5.支持条件变量：根据不同的条件唤醒不同waitSet的线程，sync只有一个waitSet。如果不用多条件变量，sync就要考虑用notifyAll，这样可能会存在一些虚假唤醒的问

**解决哲学家就餐问题：用lock.tryLock，获取不到第二把筷子就把第一把也放下。然后重试，**





### 第五章：JMM

1.可见性 ，对于频繁访问的变量，java会把主内存的变量拷贝到自己的工作内存，导致对变量的修改别的线程不可见。解决：对于这样的变量加volatile关键字，保证线程可见性。（其实sync也可以）

2.保证了可见性不一定就是原子的，它只是对于共享变量的修改让另外的线程可见罢了。 

**设计模式之犹豫：如果一个线程已经做了另一个线程准备做的事情，例如开启监控线程，设置开启标记，另一个线程发现开启标记为真则需要退出。可以利用volatile或者sync，推荐用sync，volatile不能保证原子性。**



**指令重排序**：回忆计算机组成原理，cpu流水线，为了提高程序吞吐量，cpu会重排序一些指令，让吞吐率更高，但是这样有可能引发线以下问题：

```java
boolean ready = false;
 // 线程1 执行此方法
public void actor1(I_Result r) {
 	if(ready) {
 		r.r1 = num + num;
    } 
	else {
		 r.r1 = 1;
    }
 }
 // 线程2 执行此方法
public void actor2(I_Result r) {        
	num = 2;
	 ready = true;    
}

```

有可能出现r1=0的情况：ready重排序到上面，然后执行了线程一的r.r1=0+0,所以r1出现为0的情况。

**解决：对ready加volatile**

**原理：读屏障**

开启读屏障只会不会读工作内存中的内容，会读主存。并且在都屏障之后的代码不会跨越读屏障来到读屏障前面。

**写屏障**，写屏障之前的代码不会跨越写屏障来到写屏障之后
