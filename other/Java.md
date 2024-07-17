> Edit at 2020-04-02

## Java
### 面向对象与面向过程，三大特征与五大原则
> 面向过程：问题分解成一步步用函数解决。面向对象：问题分成一个个属性，封装成对象。
> 特征：继承，封装，多态：多态就是指一个类实例的相同方法在不同情形有不同表现形式
> 原则：

### 类变量，成员变量，局部变量
> 分别在JVM的方法区，堆，栈；它们的实例对象在？
> 

### 垃圾回收机制
java中垃圾回收是系统自动调用的，可控性较差，我们可以通过调用system.gc()去调用垃圾回收，但不能保证垃圾回收一定被调用。
怎么知道哪些对象需要被回收呢？
1.引用计数法：对象有引用指向它计数器就加一，如果没有引用指向它了就说明可以被回收了，但这个不能解决相互引用的问题，所有现在基本已经不用了
2.可达性分析：系统从GCRoot开始查找引用链，如果对象没有在这个引用链上说明已经可以回收了
### JVM内存管理机制
JVM里面分为新生代、老年代和永久代，新生代一般是存放新建的对象，里面按8:1:1的比例分为一个Eden区和两个Survivor区，其中使用的是复制算法，老年代一般是存放存活时间较长的对象，也就是经过多次GC没有被回收的对象，其中使用的标记整理算法
标记清除算法:给对象加上标记直接回收，容易造成内存碎片化，现在基本不使用了
复制算法:两块同样大小的内存块，一次使用一块，GC的时候将存活的对象复制到另一块内存，每次只能使用一般的内存
标记整理算法:将存活的对象添加标记，然后移动到同连续的内存中，防止碎片化也能有效利用内存
### 类加载机制和双亲委派模型
[https://blog.csdn.net/qq_31493821/article/details/78812057](https://blog.csdn.net/qq_31493821/article/details/78812057)
类加载机制其实就是将我们的class文件加载到JVM中去，但JVM启动的时候不会一次把所以的jar文件都加载进去，而是根据需求动态加载，java中自带的有三种ClassLoader,加载顺序依次为BootStrapClassLoader、ExtClassLoader、AppClassLoader，我们自定义的默认parent是AppClassLoader（除非自己指定parent，注意：parent是一个ClassLoader的引用，并不是父类）
双亲委派模型其实就是就是ClassLoader加载一个Class的时候，如果当前类还没有被加载就会委托给他的parent去加载，直到BootStrapClassLoader，如果BootStrapClassLoader没有加载成功就又层层下放给孩子加载，这个过程其实就是一个递归的过程
### How to split a string with white space characters?
`String[] strArray = aString.split("\\s+");`

### StringBuilder & StringBuffer
可变的，线程不安全 & 线程安全， todo ...
### Equals和==的区别
==是指对内存地址进行比较 ， equals()是对字符串的内容进行比较
equals方法不能作用于基本数据类型的变量,equals继承Object类,比较的是是否是同一个对象 如果没有对equals方法进行重写,则比较的是引用类型的变量所指向的对象的
### Java怎么实现同步
使用synchronized关键字，synchronized在静态方法锁的是这个对象的class对象，在普通方法上锁的是该对象的this,在同步代码块中锁的传入的具体对象，由于同步代码块可以控制锁的粒度所以很多事时候速度更快
wait方法必须在这个同步代码块中使用，会进入等待状态并释放锁，之后调用notify或者notifyAll方法之后醒来
使用ReentrantLock（重入锁）在需要同步的代码的开始的地方调用lock(),结束的时候调用unlock()
使用volatile关键字保证可见性和防止指令重排

### CAS乐观锁以及ABA问题
CAS比较与交换，底层通常JNI实现，如果

### Wait和Sleep的区别
1)  原理不同。sleep()方法是Thread类的静态方法，是线程用来控制自身流程的，他会使此线程暂停执行一段时间，而把执行机会让给其他线程，等到计时时间一到，此线程会自动苏醒。例如，当线程执行报时功能时，每一秒钟打印出一个时间，那么此时就需要在打印方法前面加一个sleep()方法，以便让自己每隔一秒执行一次，该过程如同闹钟一样。而wait()方法是object类的方法，用于线程间通信，这个方法会使当前拥有该对象锁的进程等待，直到其他线程调用notify（）方法或者notifyAll（）时才醒来，不过开发人员也可以给他指定一个时间，自动醒来。

2)  对锁的 处理机制不同。由于sleep()方法的主要作用是让线程暂停执行一段时间，时间一到则自动恢复，不涉及线程间的通信，因此，调用sleep()方法并不会释放锁。而wait()方法则不同，当调用wait()方法后，线程会释放掉他所占用的锁，从而使线程所在对象中的其他synchronized数据可以被其他线程使用。

3)  使用区域不同。wait()方法必须放在同步控制方法和同步代码块中使用，sleep()方法则可以放在任何地方使用。sleep()方法必须捕获异常，而wait()、notify（）、notifyAll（）不需要捕获异常。在sleep的过程中，有可能被其他对象调用他的interrupt（），产生InterruptedException。由于sleep不会释放锁标志，容易导致死锁问题的发生，因此一般情况下，推荐使用wait（）方法。

### 单例、双重检查锁与指令重排
双重检查锁单例：
```java
public class Sigleton{
	public static Sigleton INSTANCE;
    private Sigleton(){
    }
    public static Sigleton getInstance(){
    	if(INSTANCE == null){
        	synchronized (Sigleton.class){
            	if(INSTANCE == null){
                	INSTANCE = new Sigleton();
                }
            }
        }
        return INSTANCE;
    }
}
```

- 双重检查解决了啥问题： 1）相对于同步锁作用在方法上，降低了性能开销；2）单个检查无法解决线程抢占得问题（会new 2次）； 
- JMM内存模型可能导致的问题：    ` INSTANCE = new Sigleton();`  指令重排会造成INSTANCE已经赋予了内存空间，不为null，但是Sigleton() 尚未初始化完成；
- 如何解决： 加入原子一致性关键字：volatile。
```java
public class Sigleton{
	public static volatile Sigleton INSTANCE;
    private Sigleton(){
    }
    public static Sigleton getInstance(){
    	if(INSTANCE == null){
        	synchronized (Sigleton.class){
            	if(INSTANCE == null){
                	INSTANCE = new Sigleton();
                }
            }
        }
        return INSTANCE;
    }
}
```

- Holder模式单例
```java
/**
 * Created by JYM on 2019/1/8
 * 单例模式：Holder方式
 * Holder的方式完全是借助了类加载的特点。
 * */

//final 不允许被继承
public final class Singleton_4
{
    //实例变量
    private byte[] data = new byte[1024];

    private Singleton_4()
    {}

    //在静态内部类中持有Singleton的实例，并且可被直接初始化；
    private static class Holder
    {
        private static Singleton_4 instance = new Singleton_4();
    }

    //调用getInstance方法，事实上是获得Holder的instance静态属性
    public static Singleton_4 getInstance()
    {
        return Holder.instance;
    }
}

/**
 * 在Singleton_4类中并没有instance的静态成员，而是将其放到了静态内部类Holder之中，因此在Singleton类的初始化过程中并不会创建
 * Singleton的实例，Holder类中定义了Singleton的静态变量，并且直接进行了实例化，当Holder被主动引用的时候则会创建Singleton的实例，
 * Singleton实例的创建过程在Java程序编译时期收集至<clinit>()方法中，该方法又是同步方法，同步方法可以保证内存的可见性、JVM指令的顺序性和原子性
 * Holder方式的单例设计是最好的设计之一，也是目前使用比较广的设计之一。
 * */
```
### 子类继承父类，关于父类是否会实例化，继承的变量在哪里分配空间
[子类继承父类的成员变量，父类对象是否会实例化？私有成员变量是否会被继承？被继承的成员变量在哪里分配空间](https://blog.csdn.net/nanruitao10/article/details/52635038)


## Java对象的生命周期

1. Create（创建阶段
2. in use（使用
3. Invisible
4. Unreachable
5. collected
6. finalized
7. deallocated

- 创建阶段

为对象分配存储空间；构造对象；从超类到子类为static成员初始化，类static在类加载的时候ClassLoader进行初始化；超类成员变量初始化；子类成员变量初始化；赋值

- 应用阶段

被强引用持有

- 不可见阶段

即使有强引用持有对象，但是这些强引用对于程序来说是**不能访问的（**accessible**）**，就会进入这个阶段（非必须经历的阶段）

- 不可达阶段

对象处于不可达阶段是指该对象不再被任何由gc root的强引用的引用链可达的状态。

- 收集阶段

当垃圾回收器发现该对象已经处于“不可达阶段”并且垃圾回收器已经对该对象的内存空间重新分配做好准备时，对象进入“收集阶段”。如果该对象已经重写了finalize()方法，并且没有被执行过，则执行该方法的操作。否则直接进入终结阶段。

- 终结阶段

当对象执行完finalize()方法后仍然处于不可达状态时，该对象进入终结阶段。在该阶段，等待垃圾回收器回收该对象空间。

- 重新分配

如果在完成上述所有工作完成后对象仍不可达，则垃圾回收器对该对象的所占用的内存空间进行回收或者再分配，该对象彻底消失。

---


## 常量池
> Class常量池，String常量池，运行时常量池


## List,set,Vector
ArrayList,LinkedList,

## HTTP和HTTPS
[这个还行](https://blog.csdn.net/xiaoming100001/article/details/81109617)
## TCP
[链接](https://blog.csdn.net/qq_38950316/article/details/81087809)
为什么需要3次握手，4次挥手？
3次握手是因为网络存在不确定性，如果客户端发起的请求被阻塞了，在这次连接断开之后才到达服务器就会重新建立连接，造成资源的浪费，而4次挥手是因为当客户端发送FIN标志到达服务器之后服务器数据可能还没有发送完毕，所以这时会先回复一个ACK给到客户端告诉他我收到了你的FIN消息，当服务器数据传输完成的时候在会发送一个FIN的标志给到客户端，所以需要4次握手
为啥客户端在收到服务器的FIN之后要等2MSL才断开？
因为一个报文在网络中的最大存活时间是1MSL，客户端发送消息给服务器之后服务器可能还会给客户端发送未收到确认消息的信号，一来一回就是2MSL，如果这2MSL没有收到服务器的消息说明服务器已经成功断开

## String最长的长度
分编译期和运行期。

## 动态代理，静态代理 和CgLib
[https://segmentfault.com/a/1190000011291179](link)

