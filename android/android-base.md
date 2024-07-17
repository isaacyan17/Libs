> Edit at 2020-06-04

## Android基础
> 零碎、或后期整理到其他目录下的知识点都可放这里
> ### 浅谈Zygote
> zygote的作用：1.启动SystemService 2.孵化应用进程
> zygote的启动流程：通过Init进程读取init.rc文件配置创建出来的，启动时会先启动虚拟机，注册JNI函数，通过jni调用ZygoteInit的main方法进入java世界，
> 在java世界中Zygote会先预加载资源，然后fork一个系统服务进程（SystemService负责启动Binder线程池和SystemServiceManager）,进入Loop循环，等待socket消息
> ### 浅谈Application启动流程
> 当我们点击一个电源时会加载一个BootLoader到RAM，然后执行这个引导程序启动系统OS，Android系统根据init.rc文件启动Init进程，Init进程通过执行init.rc文件
> 启动一个Zygote进程，Zygote会启动一个SystemService,SystemService启动了SystemServiceManager，SystemtServiceManager里面注册了一个ActivityManagerService
> ActivityManagerService启动Launcher桌面
> ### 浅谈Activity启动流程
> 
> 


TODO 为什么extends 的BaseFragment 无法获取到context / activity  （requireActivity() / getContext() /getActivity() 之类））? 

### 1. Activity 启动流程（Service，Broadcast）

#### Launcher启动App流程

- Launcher进程通过Binder将启动信息`startActivitySafely()`传递给AMS(ActivityManagerService）。（Android 8之前使用ActivityManagerProxy进程信息传递，Android8之后使用AIDL，IActivityManager）
- AMS交由ActivityStarter处理。(ActivityStarter是Android7加入的类)
- ActivityStarter处理Intent和Flag，筛选满足条件的Activity；然后由ActivityStackSupervisior处理Activity进栈的逻辑。
- ActivityStarter请求Zygote  fork新的进程，创建ApplicationThread（ApplicationThread充当应用进程与AMS通信桥梁）
- Zygote fork新进程后，创建了ActivityThread，它通过handler将callActivityOnCreate交由instrumentation处理。
> Instrumentation, 每个Activity都持有一个Instrumentation的引用。

时序图总结
1.
![](https://cdn.nlark.com/yuque/0/2021/png/1227097/1625219147255-91a36bf7-62f9-45e5-9b8b-48c547adede5.png#height=789&id=kwfCm&originHeight=789&originWidth=1198&originalType=binary&ratio=1&size=0&status=done&style=none&width=1198)

2. ![](https://cdn.nlark.com/yuque/0/2021/png/1227097/1625219164316-82a9ea15-1cfc-4fe6-870d-dfb2de68d2ed.png#height=888&id=jdTka&originHeight=888&originWidth=958&originalType=binary&ratio=1&size=0&status=done&style=none&width=958)
3. ![](https://cdn.nlark.com/yuque/0/2021/png/1227097/1625219178839-ca15f52d-3b71-4c93-8cff-22905d4fc072.png#height=817&id=GIGNV&originHeight=817&originWidth=1058&originalType=binary&ratio=1&size=0&status=done&style=none&width=1058)


#### App启动一个新的Activity
`startActivity(intent);`

-  此时Activity中已经关联了Instrumentation引用 ，从Activity->执行到`mInstrumentation.execStartActivity()；`
- `execStartActivity`交由AMS的ActivityMangerProxy代理对象执行下一步`startActivity() ` （Android不同版本，这里AMS的实现不一样）
- `ActivityMangerProxy`通过binder交由ActivityStackSupervisor
- 最终交由ActivityThread的`handleLaunchActivity`实施加载方法。
- ** Activity是在ActivityThread的performLaunchActivity方法中用ClassLoader类加载器创建出来的。**

---

##### 这里引申出2个问题

1.  应用内启动一个Activity时，Activity交由Instrumentation执行`mInstrumentation.execStartActivity()`中传入参数`mMainThread.getApplicationThread()`是一个`IApplicationThread`对象。

IApplicationThread是什么？

- 这里有2个概念，ActivityThread和ApplicationThread。ActivityThread即APP的主线程，其中包含一个内部类ApplicationThread，`ApplicationThread继承IApplicationThread.Stub`，是一个AIDL类，用作进程通讯。IApplicationThread意味着execStartActivity()传入了一个可以跨进程通讯的引用。我们梳理一下Activity启动的顺序：
```java
activity.startActivity()->
    mInstrumentation.execStartActivity()->
    	ActivityTaskManager.getService().startActivity()->
    		AcitivityTaskManagerService.startActivityAsUser()->
    			(AcitivityTaskManagerService).getActivityStartController().obtianStarter().execute()->
    				(ActivityStarter).execute()->startActivityUnchecked()->startActivityInner()->mRootWindowContainer.resumeFocusedStacksTopActivities()->
    					(其中执行了上一个activity的pause)ActivityStack.resumeTopActivityUncheckedLocked()->
    						(ActivityStackSupervisor).startSpecificActivityLocked()->realStartActivityLocked()->
    							(跨进程调用)(ApplicationThread).scheduleTransaction()->
    								(ApplicationThread).sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction)->
    									(ActivityThread内部类，继承Handler)H.EXECUTE_TRANSACTION->
    										ActivityThread.handleLaunchActivity
    						
    			
```


2. 从Android10开始，ATMS接替AMS管理组件启动，生命周期管理等。是为了降低过多职责带来的代码量过大。`RootActivityContainer`同样也是Android 10的改动。



#### Service启动过程

- [x] context.startService()

流程与activity基本类似，
```java
context.startService()->
    contextWrapper.startService()->
    	contextImpl.startService()->
    		IActivityManager.startService()->
    			(ActivityManagerService).startServiceLocked()->
    				(ActiveServices).startServiceInnerLocked()...->
    					(ApplicationThread).sendMessage()->
    						(H).CREATE_SERVICE->
    							ActivityThread.handleCreateService()->
    								
```

- [x]  context.bindService()

基本一致。
![](https://cdn.nlark.com/yuque/0/2021/png/1227097/1628236508603-d55bd883-6eac-47a2-8228-cb596c8d34b2.png#height=536&id=zwbwl&originHeight=536&originWidth=1378&originalType=binary&ratio=1&size=0&status=done&style=none&width=1378)
![](https://cdn.nlark.com/yuque/0/2021/png/1227097/1628236500160-53f84209-f3e3-4c1b-8a99-c87773e0b71b.png#height=497&id=JYqEk&originHeight=497&originWidth=1419&originalType=binary&ratio=1&size=0&status=done&style=none&width=1419)
#### ContentProvider启动过程

- [ ] TODO

#### BroadCastReceiver启动过程

- [ ] TODO


#### invalidate() 和requestLayout()的区别
invalidate会触发view的重绘，不会调用onMeasure()和onLayout()方法。onDraw()会被调用
requestLayout()会导致onMeasure()和onLayout()方法调用，但是不一定会触发onDraw()  （如果layout过程中 l,t,r,b没有改变，就可能不会触发onDraw() ）,所以有时requestLayout会和invalidate一起调用。


### 架构（MVVM、MVP）
### 设计模式
**设计模式的六大原则：**

- 单一职责 （一个类或者一个方法只负责一项职责，尽量做到类的只有一个行为原因引起变化）
- 接口隔离（一个类对另一个类的依赖应该建立在最小的接口上）
- 开闭原则（软件实体应当对扩展开放，对修改关闭）
- 依赖倒置（要面向接口编程，不要面向实现编程）
- 里式替换（继承必须确保超类所拥有的性质在子类中仍然成立）
- 迪米特原则（如果两个软件实体无须直接通信，那么就不应当发生直接的相互调用，可以通过第三方转发该调用）

**构建者模式：**
使用多个简单的对象一步一步构建成一个复杂的对象（通用的Dialog）
**原型模式：**
用原型实例指定创建对象的种类，并且通过拷贝这些原型创建新的对象（需要复用的数据）
（实现clone方法，根据原有对象模型生成一个新对象）

- 深拷贝与浅拷贝

  浅拷贝：
使用方式： 类实现Cloneable接口，并重写Clone方法。
```java
public class Student implements Cloneable{


    private int sno ;
    private String name;

    //getter ,setter 省略

    @Override
    public Object clone() throws CloneNotSupportedException {
        Student s = null;
        try{
            s = (Student)super.clone();
        }catch (Exception e){
            e.printStackTrace();
        }
        return s;
    }

}
```
```java
Student s1 = new Student();
Student s2 = s1.clone();
s1.sno == s2.sno?  true
s1==s2?  false 
拷贝出来的对象指向不同的堆空间。
但是如果Student中有对象变量（如Student.teacher），则只会拷贝变量指向内存的引用。并不会有不同的堆空间。

```
深拷贝：
类实现Serializable，通过字节流序列化与反序列化，实现拷贝。
```java
public class Student implements Serializable{

    private static final long serialVersionUID = -2232725257771333130L;

    private int sno ;
    private String name;
    private Teacher teacher;
　　//getter ,setter,toString()省略...
}

public class Teacher implements Serializable{
    private static final long serialVersionUID = 4477679176385287943L;
    private int tno;
    private String name;
　　
　//getter ,setter,toString()省略...
}

　//工具方法
　　public Object cloneObject(Object object) throws IOException, ClassNotFoundException {
        //将对象序列化
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(outputStream);
        objectOutputStream.writeObject(object);

        //将字节反序列化
        ByteArrayInputStream inputStream = new ByteArrayInputStream(outputStream.toByteArray());
        ObjectInputStream objectInputStream = new ObjectInputStream(inputStream);
        Object obj = objectInputStream.readObject();

        return obj;
    }
```

**策略模式：**
定义一系列的算法,把它们一个个封装起来, 并且使它们可相互替换。
比如数据排序（数据少的时候使用冒泡，数据多的时候使用快排）
**代理模式：**
为其他对象提供一种代理以控制对这个对象的访问。
（binder就是通过远程proxy传递参数，调用方法）
主要解决：在直接访问对象时带来的问题，比如说：要访问的对象在远程的机器上。在面向对象系统中，有些对象由于某些原因（比如对象创建开销很大，或者某些操作需要安全控制，或者需要进程外的访问），直接访问会给使用者或者系统结构带来很多麻烦，我们可以在访问此对象时加上一个对此对象的访问层。
**观察者模式：**
观察者（Observer）模式的定义：指多个对象间存在一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。这种模式有时又称作发布-订阅模式、模型-视图模式，它是对象行为型模式。
（1）Subject模块
Subjec模块有3个主要操作
addObserver()：注册添加观察者（申请订阅）
deleteObserver()：删除观察者（取消订阅）
notifyObserver()：主题状态发生变化时通知所有的观察者对象
（2）Oserver模块
Oserver模块有1个核心操作update()，当主题Subject状态改变时,将调用每个观察者的update()方法，更新通知。

- 观察者模式与发布/订阅模式

两种模式最大的区别是调度的地方。
虽然两种模式都存在订阅者和发布者（具体观察者可认为是订阅者、具体目标可认为是发布者），但是观察者模式是由具体目标调度的，而发布/订阅模式是统一         由调度中心调的，所以观察者模式的订阅者与发布者之间是存在依赖的，而发布/订阅模式则不会。
![image.png](https://cdn.nlark.com/yuque/0/2020/png/1543476/1591867592084-126c5fa5-759a-4bf4-8f67-84c58b44a311.png#height=159&id=hvILJ&originHeight=317&originWidth=692&originalType=binary&ratio=1&size=27836&status=done&style=none&width=346)![image.png](https://cdn.nlark.com/yuque/0/2020/png/1543476/1591867634866-c541c8de-2a3d-4f61-99a7-7fa8ebe50529.png#height=175&id=G07uQ&originHeight=350&originWidth=298&originalType=binary&ratio=1&size=12066&status=done&style=none&width=149)
**单例模式：**

- 懒汉式：
```java
public class A{
	private static volatile A instance;//volatile能够防止指令重排和保证变量的可见性
	public void getInstance(){
		if(A == null){
			synchronized(A.class){
				if(A == null){
					instance = new A();
				}
			}
		}
		return instance;
	}
}
```

- 饿汉式：

```java
public class A{
private static volatile A instance = new A();//注意：这里是私有的
	public void getInstance(){
		return instance ;
	}
}
```
**工厂模式：**

- **简单工厂**

每增加一个新产品都要修改工厂方法，违背了开闭原则
```java
public class Factory{
    public Product createProduct(){
        Product product = null;
        switch(type){
            case 1:
         		product  = new ProductA();
                break;
            case 2:
        		product = new ProductB();
                break;
        }
        return product;
    }
}
```

- **普通工厂**

新产品不用修改原有的方法
```java
public inteface Factory{
    Product createProduct()
｝
public class FactoryA implement Factory{
    public Product createProduct(){
    	return new ProductA();
    }
}
public class FactoryB implement Factory{
    public Product createProduct(){
    	return new ProductB();
    }
}

```

- **抽象工厂**

提供一个创建一系列相关或相互依赖对象的接口，而无需制定他们具体的类（所有的子类也得增加实现相应的接口，使用比较麻烦）
```java
public inteface Factory{
	Product createProduct()
	Gift createGift()
}
public class FactoryA implement Factory{
	public Product createProduct(){
		return new ProductA();
	}
	public GiftcreateGift(){
		return new Gift();
	}
}
public class FactoryB implement Factory{
    public Product createProduct(){
    	return new ProductB();
    }
    public GiftcreateGift(){
    	return new Gift();
    }
}
```
### 线程
**线程池：**
// 创建一个可缓存线程池
        ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
// 创建一个可重用固定个数的线程池
        ExecutorService fixedThreadPool = Executors.newFixedThreadPool(3);
//创建一个定长线程池，支持定时及周期性任务执行——延迟执行
       ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);
//创建一个单线程化的线程池
ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor();

1. 缓冲队列BlockingQueue简介：
    BlockingQueue是双缓冲队列。BlockingQueue内部使用两条队列，允许两个线程同时向队列一个存储，一个取出操作。在保证并发安全的同时，提高了队列                 的存取效率。
2. 常用的几种BlockingQueue：

   - ArrayBlockingQueue（int i）:规定大小的BlockingQueue，其构造必须指定大小。其所含的对象是FIFO顺序排序的。
   - LinkedBlockingQueue（）或者（int i）:大小不固定的BlockingQueue，若其构造时指定大小，生成的BlockingQueue有大小限制，不指定大小，其大小有Integer.MAX_VALUE来决定。其所含的对象是FIFO顺序排序的。
   - PriorityBlockingQueue（）或者（int i）:类似于LinkedBlockingQueue，但是其所含对象的排序不是FIFO，而是依据对象的自然顺序或者构造函数的Comparator决定。
   - SynchronizedQueue（）:特殊的BlockingQueue，对其的操作必须是放和取交替完成。

3. 自定义线程池（ThreadPoolExecutor和BlockingQueue连用）：
_     自定义线程池，可以用ThreadPoolExecutor类创建，它有多个构造方法来创建线程池。_
_    常见的构造函数：ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue)_

- **线程池的三种关闭**

shutdown： 线程池不再接收任务，等待线程池中所有任务完成后，关闭线程池。**常用**
shutdownNow： 线程池不再接收任务，忽略队列中的任务，尝试中断正在执行的任务，返回未执行任务列表，关闭线程池。**慎用**
awaitTermination： 线程池可以继续接收任务，当任务都完成后，或者超过设置的时间后，关闭线程池。方法是阻塞的，**考虑使用**
————————————————
版权声明：本文为CSDN博主「ITDragon龙」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：[https://blog.csdn.net/qq_19558705/article/details/79114680](https://blog.csdn.net/qq_19558705/article/details/79114680)
_为何有后端手册标注禁止 Executors._newFixedThreadPool() 和 _Executors._newCachedThreadPool()这两个方法？

```java
//newFixedThreadPool 如果核心线程始终没有释放资源，会导致LinkedBlockingQueue一直添加新的任务到阻塞队列，导致OOM
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```


```java
//newCachedThreadPool 最大线程数是Integer.MAX_VALUE，超出这个值也会报OOM的错误，但是具体错误是 unable to create new native thread.
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
### 
### 进程
#### 跨进程通信

- 跨进程通信(IPC)的几种方式
1. Intent: 进程间四大组件级的通信，只能支持Bundle类型的数据。
2. ContentProvider: 大量数据的通信，数据共享能力。
3. Socket： 常用于网络传输，传输字节流。
4. RemoteView：跨进程的UI通信方式，实现有：Notification的自定义RemoteView；桌面小组件。（本质是Binder作通信支撑）
5. Messager：未接触
6. AIDL：支持一对多的并发实时通信。
7. 文件共享

- Binder
1. 为什么设计Binder？

//TODO
Linux的通信方式有几种：信号量，管道，套接字， 共享内存，消息队列；
区别于传统的Linux的通信，Binder主要为了解决几个问题:

-  拷贝效率： 进程A->进程B，数据要从发送方用户区拷贝到内核区，再从内核区拷贝到用户区。一般历经2次拷贝
- 安全：Linux IPC一般无法获得对方的UID


### JVM、ART
### Bitmap
#### Bitmap压缩

1. 采样率压缩 （BitmapFactory.options.InSampleSize）

改变宽、高上的像素点个数，减少bitmap在内存上的占用大小

2. 质量压缩（compress）

改变bitmap占用的磁盘大小，不改变内存大小，便于图片上传。原理大概是同化了关键像素点附近的像素点，减少了颜色数量，达到减小大小的目的。

3. 缩放压缩

改变宽高比，达到图片宽高只有原图1/n的地步，减少内存占用。

4. 尺寸压缩

直接改变宽高尺寸

5. rgb_565

ARGB_8888 一个像素4个字节（颜色更丰富），rgb_565一个像素2个字节。

### 性能优化

- **布局优化**

合理选择RelativeLayout、LinearLayout、FrameLayout
1.RelativeLayout会让子View调用2次onMeasure，LinearLayout 在有weight时，也会调用子View2次onMeasure
2.RelativeLayout的子View如果高度和RelativeLayout不同，则会引发效率问题，当子View很复杂时，这个问题会更加严重。如果可以，尽量使用                padding代替margin)
去除不必要背景，getWindow().setBackgroundDrawable(null)
使用include(相同布局复用)、merge（减少布局嵌套）、viewstub（布局需要时加载）标签
使用FlexBoxLayout代替LinearLayout（减少布局嵌套）
使用ConstraintLayout代替RelativeLayout（减少布局嵌套）
用SurfaceView或TextureView代替普通View（子线程加载布局）

- **电量优化**
- **流量优化**
- **内存优化**

算法优化
使用线程池管理线程，避免线程的新建
子线程加载耗时操作，延迟加载耗时操作
资源释放（广播注销，文件流关闭，数据库关闭等等）    
使用 静态内部类+WeakReference 代替内部类(内部类持有外部内的引用 如：handle内存泄漏)
单例模式内存泄漏（单例使用context.getApplication()或者记得释放context）
使用缓存（LruCache的原理、缓存主要包括对象缓存、IO缓存、网络缓存、DB缓存，对象缓存能减少内存的分配，IO缓存减少磁盘的读写次数，网络缓存减少网         络传输，DB缓存较少Database的访问次数）
数据存储优化（用StringBuilder、StringBuffer代替String）
使用webView，在Activity.onDestory需要移除和销毁，webView.removeAllViews()和webView.destory()
### 事件分发
**分发过程**
从驱动层到应用层，一般是分三个步骤来解释

- touch事件从驱动层传递到framework的inputManagerService
- WMS通过ViewRoolmple传递到窗口
- touch事件到达DecorView后，一层层传递到目标view。

#### InputManagerService
当我们手指触摸屏幕时，Android硬件会引起中断事件，通过输入设备的驱动模块处理硬件引发的中断，
```java
include/linux/interrupt.h

/**
 * request_irq - Add a handler for an interrupt line
 * @irq:	The interrupt line to allocate
 * @handler:	Function to be called when the IRQ occurs.
 *		Primary handler for threaded interrupts
 *		If NULL, the default primary handler is installed
 * @flags:	Handling flags
 * @name:	Name of the device generating this interrupt
 * @dev:	A cookie passed to the handler function
 *
 * This call allocates an interrupt and establishes a handler; see
 * the documentation for request_threaded_irq() for details.
 */
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)
{
	return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}

```
request_irq()函数注册 request_threaded_irq()函数处理中断信息，最终将信息存入/dev/input/eventX文件中。
原始触摸事件已经收集完毕，接下来就需要system_server进程处理、加工我们的触摸事件，然后传递给应用层window了。
![image.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591947416674-456dfc1f-613f-491b-a1d1-53a4233b5b00.png#height=655&id=AkQdO&originHeight=655&originWidth=882&originalType=binary&ratio=1&size=75663&status=done&style=none&width=882)
system_server是Android系统的核心内容之一，非常多的服务运行在这个进程中。（计算机时间起始在1970年也是在systemServer的run方法中初始化的）我们关注的触摸事件相关的服务：InputManagerService也是在system_server进程中启动的。
InputMangerService初始化时会创建几个关键对象：EventHub，InputReader，InputDispatcher。下面这张图清晰的展示了这几个对象和处理、传递触摸事件这些事务的关系。
![image.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591947959816-764c8622-8ef8-4270-bc51-36061340c7d7.png#height=637&id=xZGrS&originHeight=637&originWidth=873&originalType=binary&ratio=1&size=93287&status=done&style=none&width=873)

我们来分析下这个3个InputManagerService中创建的对象。

##### EventHub
EventHub的作用是对/dev/input/eventX中的文件进行监听和读取，并将其封装成RawEvent结构体，传递给InputReader使用。
如何监听、读取？
```cpp
EventHub::EventHub(void) :
    mBuiltInKeyboardId(NO_BUILT_IN_KEYBOARD), mNextDeviceId(1), mControllerNumbers(),
    mOpeningDevices(0), mClosingDevices(0),
    mNeedToSendFinishedDeviceScan(false),
    mNeedToReopenDevices(false), mNeedToScanDevices(true),
    mPendingEventCount(0), mPendingEventIndex(0), mPendingINotify(false) {
          ...
        mEpollFd = epoll_create(EPOLL_SIZE_HINT);    //创建epoll
        mINotifyFd = inotify_init();     // 初始化监听
        int result = inotify_add_watch(mINotifyFd, DEVICE_PATH, IN_DELETE | IN_CREATE);  //DEVICE_PATH就是"/dev/input"路径
        result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mINotifyFd, &eventItem);     //将INotifyFd添加到epoll
        ...
}
```
##### InputReader
InputReader是一个线程，通过InputManager的initialize初始化创建，用EventHub的getEvent()方法调取EventHub包装好的输入事件并处理，然后交给InputDispatcher。其内部是通过loopOnce()方法来不断读取事件。
![image.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591951395808-4273430d-7d23-4475-be33-9dbf66fb3029.png#height=616&id=ye0Xu&originHeight=616&originWidth=777&originalType=binary&ratio=1&size=36776&status=done&style=none&width=777)
第二个红框：当读取了EventHub事件，调用processEventsLocked()，这个方法会执行一系列判断后，最终执行process函数：

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591951582265-c659b67b-d44c-4c4b-99c0-7f6d61348e95.png#height=566&id=pDgFl&originHeight=566&originWidth=788&originalType=binary&ratio=1&size=40387&status=done&style=none&width=788)
用InputMapper处理rawEvent。这里需要说明的是，不同的rawEvent会由不同的Mapper去处理，比如触摸事件，会由TouchInputMapper处理，最终分发给InputDispatcher。

##### InputDispatcher
InputDispatcher同样也是一个线程，同样的，由InputManager的initialize初始化。InputDispatcher的工作主要是接收InputReader的输入事件，并派发给合适的窗口处理。
这里说明一下，InputDispatcher和InputReader两个线程都是android.display线程：

> android.display线程：属于Looper线程，用于处理Java层的IMS.InputManagerHandler和JNI层的NativeInputManager中指定的MessageHandler消息;


我们在InputDispatcher的分发里可以了解它是怎么找到需要处理触摸事件的window的：
```cpp
int32_t injectionResult;
if (isPointerEvent) {
// Pointer event.  (eg. touchscreen)
	injectionResult = findTouchedWindowTargetsLocked(currentTime,
	entry, inputTargets, nextWakeupTime, &conflictingPointerActions);
} else {
// Non touch event.  (eg. trackball)
	injectionResult = findFocusedWindowTargetsLocked(currentTime,
                entry, inputTargets, nextWakeupTime);
}
```
对触摸事件做一个判断，然后调用findTouchedWindowTargesLocked()函数，取出entry里面的pointerIndex与触摸点坐标的x y值，从前向后遍历所有的window。
```cpp
findTouchedWindowTargetsLocked 函数：

1210  // Traverse windows from front to back to find touched window and outside targets.
1211        size_t numWindows = mWindowHandles.size();
1212        for (size_t i = 0; i < numWindows; i++) {
1213            sp<InputWindowHandle> windowHandle = mWindowHandles.itemAt(i);
1214            const InputWindowInfo* windowInfo = windowHandle->getInfo();
1215            if (windowInfo->displayId != displayId) {
1216                continue; // wrong display
1217            }
1218
1219            int32_t flags = windowInfo->layoutParamsFlags;
1220            if (windowInfo->visible) {
1221                if (! (flags & InputWindowInfo::FLAG_NOT_TOUCHABLE)) {
1222                    isTouchModal = (flags & (InputWindowInfo::FLAG_NOT_FOCUSABLE
1223                            | InputWindowInfo::FLAG_NOT_TOUCH_MODAL)) == 0;
1224                    if (isTouchModal || windowInfo->touchableRegionContainsPoint(x, y)) {
1225                        newTouchedWindowHandle = windowHandle;
1226                        break; // found touched window, exit window loop
1227                    }
1228                }
1229
1230                if (maskedAction == AMOTION_EVENT_ACTION_DOWN
1231                        && (flags & InputWindowInfo::FLAG_WATCH_OUTSIDE_TOUCH)) {
1232                    mTempTouchState.addOrUpdateWindow(
1233                            windowHandle, InputTarget::FLAG_DISPATCH_AS_OUTSIDE, BitSet32(0));
1234                }
1235            }
1236        }

// 简单总结就是：displayId是否匹配(1216行); 可见的window是否能被触摸(1224行)； 触摸的区域是否是window之外的区域(1230行)
```
找到了window，前期的准备工作终于告一段落，接下来，我们只需将触摸事件传给应用层，即对应的window就好了。如何与window通信呢？
![image.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591953346715-64053bae-edde-42e9-ad02-fd6dae8f09a3.png#height=641&id=chq92&originHeight=641&originWidth=888&originalType=binary&ratio=1&size=98550&status=done&style=none&width=888)

答案就是使用socket。
在InputDispatcher 向目标window分发事件时有这一段：
```cpp
getConnectionIndexLocked(inputTarget.inputChannel);
```
inputChannel包含了一个socket用于跨进程通信，通过sendMessage()和receiveMessage()两个函数实现。

到这，我们可以简单总结一下Input事件从硬件到IMS的流程了：
Linux kernel中断 ->InputManagerService服务处理（主要由EventHub，InputReader和InputDispatcher处理）->通过InputChannel（socket）->到达WindowManagerService。

#### WMS
前面我们说到，input事件从硬件，经过IMS处理，由socket传递到WindowManagerService，到这里，原始的Input事件已经传递到应用的UI线程中，接下来只需要将这个事件传递给ViewRootImpl，在线程中，是通过looper机制来实现分发的。
我们的input事件是从native传到java层的，在looper中的messageQueue中，
```java
Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        nativePollOnce(ptr, nextPollTimeoutMillis);
        ...
    }
}
```
`next()`方法中调用了`nativePollOnce()`去处理native中的事件。它会将`InputDispatcher`分发过来的event进行一系列转化，最终来到java层的`dispatchInputEvent()`方法进行分发。
```java
private void dispatchInputEvent(int seq, InputEvent event) {
    mSeqMap.put(event.getSequenceNumber(), seq);
    onInputEvent(event);
}
```
`InputEventReceiver`是一个抽象类，具体实现类是`ViewRootImpl`的内部类`WindowInputEventReceiver`，它覆盖了`onInputEvent()`方法：
```java
@Override
public void onInputEvent(InputEvent event) {
    enqueueInputEvent(event, this, 0, true);
}
```
最终，
```java
public final boolean dispatchPointerEvent(MotionEvent event) {
    if (event.isTouchEvent()) {
        return dispatchTouchEvent(event);
    } else {
        return dispatchGenericMotionEvent(event);
    }
}
```
事件在`dispatchPointerEvent()`中传递到了dispatchTouchEvent，这也是view层分发机制的开始。


#### 应用层分发
一般讨论ViewGroup和View的分发机制。

- ViewGroup 是View的集合，内部可能包含多个子View；手指触摸时，我们要分析以下几个方面：

1）当前Group是否要拦截touch事件。
2）是否要分发touch事件给子View。
3）怎样进行分发。

- View是视图控件的一个基本单位，不能再细分了，也不包含其他的子View。所以它是事件分发机制最终到达的地方。View主要处理以下几点：

1） 是否消费传递过来的touch事件。（消费逻辑在onTouchEvent）
2) 是否存在onTouchListener。

当touch事件到达ViewGroup后，会先在dispatchTouchEvent中判断是否拦截该事件，不拦截则进行遍历分发，抵达ViewGroup所包含的子View，子View的dispatchTouchEvent判断是否要消费这个事件，如果消费则返回true，并交由子View的onTouchEvent去处理。
注意，这里的过程是假设用户没有注册onTouchListener的监听的，如果用户注册了触摸事件的监听，那么onTouchListener处理touch事件的调度顺序是最高的。
###### ViewGroup的dispatchTouchEvent
ViewGroup的dispatchTouchEvent主要有三个步骤：

- 1. 是否拦截事件
- 2. 向子View分发事件
- 3. mFirstTouchTarget再次分发。第三步主要判断第二步分发是否有子View消费。如果没有子View消费，则将调用自身的onTouchEvent方法处理事件。如果有子view消费，会将mFirstTarget指向子View。那么后续事件就交由mFirstTarget指向的View去处理。

第一步：
> 是否拦截事件


![image.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591799964200-aa503b6a-f3fe-4ff8-9404-367138b1b3dd.png#height=281&id=tCejv&originHeight=562&originWidth=1664&originalType=binary&ratio=1&size=112696&status=done&style=none&width=832)
如果事件是ACTION_DOWN，则调用onInterceptTouchEvent方法判断是否要进行拦截。
如果mFirstTouchTarget不为空，则表示已经绑定了消费touch事件的子View，同样调用onInterceptTouchEvent方法判断是否要进行拦截。

第二步：
>  向子View分发事件

如果不做拦截，分发的判断条件是：
![image.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591800589820-b181ccf8-a9c3-475d-946d-ba8997d476b6.png#height=258&id=GNTmn&originHeight=516&originWidth=1284&originalType=binary&ratio=1&size=108111&status=done&style=none&width=642)
canceled我们后面用另外一个场景描述； 这里if表示即没有取消，也不做拦截的话，就将事件分发给子View：
![image.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591801203454-89e4a82f-ab20-4cec-8a77-05ef6e7326fc.png#height=349&id=w36KR&originHeight=698&originWidth=1820&originalType=binary&ratio=1&size=135440&status=done&style=none&width=910)
可以看到分发给子的方式是遍历自己包含的所有子View。其中遍历过程中还有一个判断条件：
![image.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591801649880-ee499d65-3847-45a7-8587-ab73cc5a5852.png#height=122&id=VnBrS&originHeight=244&originWidth=1760&originalType=binary&ratio=1&size=40084&status=done&style=none&width=880)如果子View的坐标范围不在touch点的坐标内，或者子View并没有在动画状态，则不将touch事件分发给该子View。

如果子View消费该事件，则将mFirstTouchTarget赋值给子View。

第三步：
> mFirstTouchTarget再次分发


我们在第二步分发遍历的子View的时候，如果找到子View消费这个事件，那么mFirstTouchTarget为空，调用自身的super.dispatchTouchEvent方法-自身的onTouchEvent方法处理事件。
![image.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591802712571-da23edf7-41e0-4ed0-870e-369f6c5c26d7.png#height=415&id=s3rwo&originHeight=830&originWidth=1934&originalType=binary&ratio=1&size=170697&status=done&style=none&width=967)
到这里我们能总结一下mFirstTouchTarget的作用：
记录谁（子View）消费了touch事件（主要是DOWN事件）。

在源码中我们能看到注释mFirstTouchTarget是一个链表结构TouchTarget。为什么是一个链表呢？是为了方便设备的多指操作。每一个指头的touch事件都能用链表TouchTarget保存。

###### CANCEL事件
> 场景：Scrollview中包含子View，在接收DOWN事件时，Scrollview本身不拦截，事件会传递给其包含的子View。当滑动一定距离后，scrollview本身onInterceptTouchEvent会返回true，表示拦截事件，并开始滚动。


此时子view是否还会继续接收后续的事件呢？
答：不会再处理后续事件。
我们先假设子View会消费DOWN事件，我们前面提到过，如果子view有消费DOWN事件，那么父view的mFirstTouchTarget会赋值子VIew。也就是说不为空了。
![image.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591844698818-17fb302e-dcb5-4b3d-befe-c8b1bed0f4da.png#height=818&id=LLF7u&originHeight=818&originWidth=967&originalType=binary&ratio=1&size=78247&status=done&style=none&width=967)

在ViewGroup的dispatchTouchEvent方法第三步中，mFirstTouchTarget!=null 时，我们看到还有一段逻辑：
  

```
      final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            ....
                        }
```
如果cancelChild==true，我们看看dispatchTransformedTouchEvent做了什么处理：
![image.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591844994619-755d9a9b-c8c7-4ccb-a8d0-54f3762886df.png#height=538&id=dPa9E&originHeight=538&originWidth=820&originalType=binary&ratio=1&size=54825&status=done&style=none&width=820)
也就是说cancel==true时，给子View传递了ACTION_CANCEL这个事件。当子View收到这个事件以后，就会丢失焦点，也即不处理后续点击事件了。

到这里，我们将ViewGroup的分发方法简述了一遍，其实事件分发过程，dispatchTouchEvent方法就是核心思路，View的处理也大同小异。


## Android应用
### 蓝牙（低功耗、双模）（HAL层）
### WIFI（热点，AP通信，局域网实现）
### Camera（扫码，拍照）（HAL层）
### Media Player（多媒体）
### Webview（交互）
#### webview与原生的相互通信

---

##### 原生调用JS

- 通过webview的`loadUrl()`；
- webview的`evaluateJavascript()`;
##### JS调用原生

- 通过webview的`@addJavascriptInterface`进行映射
- 通过webviewClient的`shouldOverridingUrlLoading()`方法进行拦截url
- 通过webChromeClient的`onJsAlert()`,`onJsConfirm()`,`onJsPrompt`去拦截JS对话框消息

---


- native调用js

1. 允许执行JavaScript代码`webView.setJavaScriptEnabled(true);`
2. loadUrl；`webView.loadUrl();`(在onPageFinished后调用能正常执行)

- js调用native
1. `webView.addJavascriptInterface(new MyJavascriptInterface(this), "injectedObject");`
```java
public class MyJavascriptInterface {
    @JavascriptInterface
    public void sayHello() {
        Log.e("hello tom");
    }

    @JavascriptInterface
    public void sayHello(String name) {
        Log.e("hello" + name);
    }
    
    @JavascriptInterface
    public void printImageSrc(String src) {
        Log.e("src", src);
    }
}
```

2. JavaScript调用原生方法，`window.injectedObject.sayHello()`

- 双向调用：`webview.evaluateJavascript()`

![image.png](https://cdn.nlark.com/yuque/0/2021/png/1227097/1637052978675-5bcc4d6d-6368-4ce2-91ba-147f8364e120.png#clientId=u9059665d-bf7c-4&from=paste&id=u9b8d7874&originHeight=420&originWidth=1189&originalType=url&ratio=1&size=96943&status=done&style=none&taskId=u1bd842ca-1187-48c1-b05f-f0af66420d1)

#### webview的安全性问题

- `addJavaScriptInterface`

在4.4以下的版本中，` webView.addJavascriptInterface(new JSObject(), "myObj");`的方法会导致js能获取到一个Android的原生对象，通过反射的方式，可以甚至拿到‘Runtime’类，从而进行任意代码的执行。
4.4以下的解决办法：通过重写`addJavascriptInterface`方法，拦截onJsPrompt()方法，约定安全的js信息，然后再调用安卓的原生方法。

- 密码存储问题

webview默认开启密码保存功能：`mWebView.setSavePassword(true)`，存储的密码会在'/data/data/packageName/webview.db'中。
解决办法：关闭密码保存功能。

- 域控制

[reference](https://www.jianshu.com/p/3a345d27cd42)

---


#### webview的内存泄露问题
> 内存泄露的诱因是webview持有了一个Activity的引用，如果没有显示地调用`parent.removeView(Webview)`,并接着调用`webView.onDestroy()`的话，就无法移除这个引用。[reference](https://juejin.cn/post/6901487965562732551)

内存泄露的问题在高版本安卓上已经完善修复了。4.4+的安卓已经将webView使用Chrome开源内核。5.0+将webView迁移到一个独立的app；即使安卓版本低，但并不代表webView，即chromium内核版本低。

**问题的产生**：低内核版本的webview，在`onAttachToWindow`的方法中注册了一个`ComponentCallbacks`，这个componentCallback持有了一个上下文的引用，也就是xml webview中的上下文，即Activity。
在`onDettachFromWindow`中会解注册。
webview的`onDestroy`方法中，并没有优先执行`onDettachFromWindow`这个方法，导致解注册的条件没有执行，最终产生了泄露。

**新旧版本对比**：
```java
//old 54.0.2805.1

public void destroy() {
    if (isDestroyed(NO_WARN)) return;
    // Remove pending messages
    mContentsClient.getCallbackHelper().removeCallbacksAndMessages();
    // ...
    if (mIsAttachedToWindow) {
        Log.w(TAG, "WebView.destroy() called while WebView is still attached to window.");
        nativeOnDetachedFromWindow(mNativeAwContents);
    }
    mIsDestroyed = true;
}

public void onAttachedToWindow() {
    if (isDestroyed(NO_WARN)) return;
    // ...
    if (mComponentCallbacks != null) return;
    mComponentCallbacks = new AwComponentCallbacks();
    mContext.registerComponentCallbacks(mComponentCallbacks);
}

public void onDetachedFromWindow() {
    if (isDestroyed(NO_WARN)) return;
    nativeOnDetachedFromWindow(mNativeAwContents);
    // ...
    if (mComponentCallbacks != null) {
        mContext.unregisterComponentCallbacks(mComponentCallbacks);
        mComponentCallbacks = null;
    }
}

```

```java
//new 86.0.4240.198


public void destroy() {
    if (isDestroyed(NO_WARN)) return;
    // ...
    // Remove pending messages
    mContentsClient.getCallbackHelper().removeCallbacksAndMessages();
    if (mIsAttachedToWindow) {
        // 如果此时没有 detach 则先调用 onDetachedFromWindow 方法，然后才将 mIsDestroyed 置为 true
        Log.w(TAG, "WebView.destroy() called while WebView is still attached to window.");
        onDetachedFromWindow();
    }
    mIsDestroyed = true;
}

// onAttachedToWindow 时会调用
public void onAttachedToWindow() {
    if (isDestroyed(NO_WARN)) return;
    if (mIsAttachedToWindow) {
        Log.w(TAG, "onAttachedToWindow called when already attached. Ignoring");
        return;
    }
    mIsAttachedToWindow = true;
    // ...
    if (mComponentCallbacks != null) return;
    mComponentCallbacks = new AwComponentCallbacks();
    // 注册 ComponentCallbacks
    mContext.registerComponentCallbacks(mComponentCallbacks);
}

// onDetachedFromWindow 时会调用
public void onDetachedFromWindow() {
    if (isDestroyed(NO_WARN)) return;
    if (!mIsAttachedToWindow) {
        Log.w(TAG, "onDetachedFromWindow called when already detached. Ignoring");
        return;
    }
    mIsAttachedToWindow = false;
    // ...
    if (mComponentCallbacks != null) {
        // 将 ComponentCallbacks 解注册
        mContext.unregisterComponentCallbacks(mComponentCallbacks);
        mComponentCallbacks = null;
    }
}

```



## 三方框架
### 网络（okhttp，retrofit，rxjava）
![OkHttpClient.Builder().png](https://cdn.nlark.com/yuque/0/2020/png/1543476/1591871482283-0d95fd8e-ced9-4b56-b977-a0a06b78f77b.png#height=2028&id=THVo9&originHeight=2028&originWidth=1865&originalType=binary&ratio=1&size=134962&status=done&style=none&width=1865)
#### Okhttp

---

OKHttp的基础用法：
```java
// 创建client客户端
OkHttpClient client = new OkHttpClient();
// 创建request
Request request = new Request.Builder()
        .url(url)
        .build();
// 执行请求任务
Response response = client.newCall(request).execute();
```
##### 同步请求

- 创建request采用builder模式（builder：便于复杂多变的初始化）。OKhttpClient也是构建者模式（builder。
- `newCall(request)`中执行newRealCall()方法。直到execute()方法,调用责任链`Response result = getResponseWithInterceptorChain();`拿到返回值response.
- 同步阻塞当前线程, 需要在非主线程中调用.
- 责任链可以做到链式调用: `chain.proceed()` (原因: 所有的拦截器Interceptor实现了统一的接口, 实现proceed方法, 获取到当前拦截器的Index, 然后在任意位置拦截并传递下一个拦截器对象. `interceptor.intercept(next)`)

##### 异步请求

- 请求任务加入队列`newCall(request).enqueue(). `(问题: enqueue结构 , 如何实施队列移动)

---

> 请求对象转化成AsyncCall加入队列后(双端队列Deque; ) ,  enqueue方法中判断: 是否超过最大连接数64; 是否超过单次最大主机连接数5? 均没超过时执行运行队列`runningAsyncCalls`的excute方法执行请求。否则加入等待队列`readyAsyncCalls`。队列均由Dispatcher触发执行next。


- 后续步骤和同步类似。


#### RxJava

---




### AOP技术（框架、埋点，可以参考[DoraemonKit](https://github.com/didi/DoraemonKit) 的AOP实现）
### 图片（Glide）
### 热修复、组件化与插件化

- **类加载机制和双亲委派模型**

[https://blog.csdn.net/qq_31493821/article/details/78812057](https://blog.csdn.net/qq_31493821/article/details/78812057)
类加载机制其实就是将我们的class文件加载到JVM中去，但JVM启动的时候不会一次把所以的jar文件都加载进去，而是根据需求动态加载，java中自带的有三种ClassLoader,加载顺序依次为BootStrapClassLoader、ExtClassLoader、AppClassLoader，我们自定义的默认parent是AppClassLoader（除非自己指定parent，注意：parent是一个ClassLoader的引用，并不是父类）
双亲委派模型其实就是就是ClassLoader加载一个Class的时候，如果当前类还没有被加载就会委托给他的parent去加载，直到BootStrapClassLoader，如果BootStrapClassLoader没有加载成功就又层层下放给孩子加载，这个过程其实就是一个递归的过程
```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先，检测是否已经加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        //父加载器不为空则调用父加载器的loadClass
                        c = parent.loadClass(name, false);
                    } else {
                        //父加载器为空则调用Bootstrap Classloader
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
 
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    //父加载器没有找到，则调用findclass
                    c = findClass(name);
 
                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                //调用resolveClass()
                resolveClass(c);
            }
            return c;
        }
    }
```

- **热修复**

[https://www.jianshu.com/p/e179fcc97666](https://www.jianshu.com/p/e179fcc97666)
我们先来说说代码修复，Andorid中动态加载代码的ClassLoader有PathClassLoader和DexClassLoader，其中PathClassLoader在Dalvik虚拟机中只能用来加载已安装apk的类，而DexClassLoader在Dalvik和ART虚拟机中都能加载未安装apk或者dex中的类，所以热修复使用DexClassLoader来加载补丁包中的类。DexClassLoader重写了findClass方法去遍历查找文件中的class对象，找到第一个就return掉，所以我们只需要创建一个同名的Class加入到数组的最前面就可以实现代码的热修复效果
```java
public class BaseDexClassLoader extends ClassLoader {
    // ...
    
    // dex文件的路径列表
    private final DexPathList pathList;
    
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
            for (Throwable t : suppressedExceptions) {
                cnfe.addSuppressed(t);
            }
            throw cnfe;
        }
        return c;
    }
    // ...
}
final class DexPathList {
    // ...
    
    // 每个元素代表着一个dex
    private Element[] dexElements;
    
    public Class<?> findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            Class<?> clazz = element.findClass(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }

        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
    
    // ...
}
public class HotfixHelper {
    
        public static void applyPatch(Context context) {
        // 获取宿主的ClassLoader
        ClassLoader classLoader = context.getClassLoader();
        Class loaderClass = BaseDexClassLoader.class;
        try {
            // 获取宿主ClassLoader的pathList对象
            Object hostPathList = ReflectUtil.getField(loaderClass, classLoader, "pathList");
            // 获取宿主pathList对象中的dexElements数组对象
            Object hostDexElement = ReflectUtil.getField(hostPathList.getClass(), hostPathList, "dexElements");

            File optimizeDir = new File(context.getCacheDir() + "/optimize");
            if (!optimizeDir.exists()) {
                optimizeDir.mkdir();
            }
            // 创建补丁包的类加载器
            DexClassLoader patchClassLoader = new DexClassLoader(context.getCacheDir() + "/patch.dex", optimizeDir.getPath(), null, classLoader);
            // 获取补丁ClassLoader中的pathList对象
            Object patchPathList = ReflectUtil.getField(loaderClass, patchClassLoader, "pathList");
            // 获取补丁pathList对象中的dexElements数组对象
            Object patchDexElement = ReflectUtil.getField(patchPathList.getClass(), patchPathList, "dexElements");

            // 合并宿主中的dexElements和补丁中的dexElements，并把补丁的dexElements放在数组的头部
            Object newDexElements = combineArray(hostDexElement, patchDexElement);
            // 将合并完成的dexElements设置到宿主ClassLoader中去
            ReflectUtil.setField(hostPathList.getClass(), hostPathList, "dexElements", newDexElements);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    /**
     *
     * @param hostElements    宿主中的dexElements
     * @param patchElements   补丁包中的dexElements
     * @return Object         合并成的dexElements
     */
    private static Object combineArray(Object hostElements, Object patchElements) {
        Class<?> componentType = hostElements.getClass().getComponentType();
        int i = Array.getLength(hostElements);
        int j = Array.getLength(patchElements);
        int k = i + j;
        Object result = Array.newInstance(componentType, k);
        // 将补丁包的dexElements合并到头部
        System.arraycopy(patchElements, 0, result, 0, j);
        System.arraycopy(hostElements, 0, result, j, i);
        return result;
    }
}
```

- **组件化**
### 事件总线（EventBus）
EventBus是典型的发布/订阅事件模式，EventBus使用很简单EventBus.getDefault().register(this),先通过单例模式获取一个EventBus对象，然后将当前的类中注册给订阅用户（拿到当前类的class对象，获取public修饰的，有@Subscribe注解的，参数个数只有一个的所有方法添加到订阅中，将所有的订阅方法加入缓存方便下次查找，所谓的订阅其实是3个Map，一个以class为key的Map，解注册的时候根据这个来遍历，一个是以event为Key的Map（subscriptionsByEventType），发送事件的时候查找所有相同事件的订阅，还有一个粘性的Map，处理粘性事件，在postSticky()的时候添加，接着就是事件的发送了，事件发送会拿到当前线程的postingState对象，将要发送的事件添加到该对象的队列中，该队列通过subscriptionsByEventType拿到所有的订阅方法，通过方法上不同的ThreadMode将该方法同反射的方式在不同的线程上执行（_POSTING：直接反射执行方法， MAIN：发送线程在主线程直接反射执行方法的订阅否则通过handler发送到主线程之后再反射执行订阅，BACKGROUND：在子线程直接执行订阅主线程添加到线程池中执行，ASYNC：添加到线程池中执行_）
```java
public void register(Object subscriber) {
    	//获取注册对象的class
        Class<?> subscriberClass = subscriber.getClass();
    	//找到当前类的所有订阅方法（public修饰、@Subscribe修饰、参数有且只有一个的方法）
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            //将所有上述方法添加到订阅中
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
//查找当前类的所有订阅方法
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
    	//首先从缓存中找
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
		// 由于使用了默认的EventBusBuilder，则ignoreGeneratedIndex属性默认为false，即是否忽略注解生成器
        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
    
        if (subscriberMethods.isEmpty()) {//没找到订阅方法抛出异常
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {//找到添加到缓存中
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);
            }
            //移动父类
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

//从反射中找
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        FindState findState = prepareFindState();//享元模式复用FindState
        findState.initForSubscriber(subscriberClass);
    	//循环查找自己和所有父类订阅方法
        while (findState.clazz != null) {
            findUsingReflectionInSingleClass(findState);
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

    private void findUsingReflectionInSingleClass(FindState findState) {
        //获取到类的所有方法
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            //getDeclaredMethods比getMethods获取方法的速度更快，特别是这个类很复杂的时候
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        //找到public修饰、@Subscribe修饰、参数有且只有一个的方法
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        //检测该方法是否可以添加到了订阅中
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    //只能有一个参数
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                //必须是public，不能是static，不能是abstract
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    	//获取当前方法的参数
        Class<?> eventType = subscriberMethod.eventType;
    	//根据和和方法创建一个新的订阅
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    	//根据参数类型，拿到所有类中对应该参数的订阅
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {//还没有改参数的订阅，new一个List来接收订阅，把这个list添加到subscriptionsByEventType中
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

    	//根据优先级添加，优先相同和或者最小则添加到最后
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

    	//添加到typesBySubscriber中，typesBySubscriber根据class查找订阅
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);

    	//粘性事件处理
        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```
![20190812224935679.png](https://cdn.nlark.com/yuque/0/2020/png/1543476/1593568632717-16d3f51f-fb84-45bd-834e-1f369587def7.png#height=1312&id=rQKJ3&originHeight=1312&originWidth=854&originalType=binary&ratio=1&size=139269&status=done&style=none&width=854)
![20190812224950815.png](https://cdn.nlark.com/yuque/0/2020/png/1543476/1593568647643-30c214e1-b58b-4011-9e18-1e247bbc98f1.png#height=1490&id=UccvZ&originHeight=1490&originWidth=688&originalType=binary&ratio=1&size=122520&status=done&style=none&width=688)
![20190812225005772.png](https://cdn.nlark.com/yuque/0/2020/png/1543476/1593568663187-85c00544-2f0b-4934-8e02-d93791c64a42.png#height=1175&id=vovPL&originHeight=1175&originWidth=609&originalType=binary&ratio=1&size=90371&status=done&style=none&width=609)

# Android系统
## 核心服务
### ActivityManager

- activity
- service
- broadcast
- provider
### WindowManager

- WindowManager启动
- 图形系统
- surface
- ...
### PackageManager
### TODO: PowerManager 、InputManager、SensorManager。。。

## 系统启动过程

> init 、zygote、systemSever、serviceManager

![微信图片_20200519090554.png](https://cdn.nlark.com/yuque/0/2020/png/1227097/1591237220839-ece50cad-58a9-4fe9-b80a-467e6734da8a.png#height=1324&id=FyNgE&originHeight=1324&originWidth=1080&originalType=binary&ratio=1&size=242525&status=done&style=none&width=1080)

## 内核技术
### CPU调度
### 进程管理
### 文件系统
### 内存管理
## 通信方式
### Binder

- **IPC注意事项**

 1.单例和静态失效
 2.sharePrefences不在可靠
 3.线程同步失效
 4.Application会多次创建

- **IPC通讯方式**

     文件共享，广播，contentPrivoder,socket，binder（messagenr，AIDL）

- **为啥使用binder**

 安全：传统的通讯UID/PID通过数据包传输，安全性完全由上层应用控制，binder的标志由IPC机制通过内核添加
 稳定：基于c/s架构，分工明确
 性能：不用像socket两次内存拷贝 [用户空间(server)->内核空间->用户空间(client)]，
也不像内存共享，虽然速度快，但是难以控制，
 	binder一次拷贝就行[用户空间（service）->内核空间(client用户空间与内核空间指向同一个地方)]

- **binder的原理**

 binder是Android跨进程通讯中最核心的类，说到跨进程通讯首先我们要知道跨进程通讯的原理，每个进程都有一个用户空间和一个内核空间，用户空间不同进程是不能共享的，而内核空间可以共享，所以跨进程通讯其实就是共享内核空间来进行通讯（具体通过binder驱动实现）。
binder通讯包括四个主要模块，Client、Service（包含一个binder）、ServiceMananger、binder驱动（前三个模块分别属于3个不同的进程，都要通过binder驱动通讯，ServiceMananger是一个编号为0的binder，所以默认支持跨进程通讯），首先Service在ServiceMananger注册，Client通过服务名称获取到Service的代理对象，binder驱动将client数据打包，调用service相同的方法
![2018090305451219.jpg](https://cdn.nlark.com/yuque/0/2020/jpeg/1543476/1592209216837-2530c262-7c94-4732-90c3-588a270861ab.jpeg#height=449&id=NC4en&originHeight=449&originWidth=866&originalType=binary&ratio=1&size=36695&status=done&style=none&width=866)

---

#### binder原理补充

- 进程可以分为内核态和用户态。一个用户进程如果需要访问系统资源，比如文件，就需要经过**系统调用。**通过系统调用 就可以使进程从用户态转换成内核态，处理器交由内核代码执行。
- binder驱动是负责各用户进程通过binder进行通信的**内核模块**。binder驱动并不是一开始写在内核空间的代码，而是使用Linux的**动态可加载模块（Loadable Kernel Module）**的机制实现的。它运行时被链接到内核空间作为内核的一部分来执行。
- [Binder](http://weishu.me/2016/01/12/binder-index-for-newer/)

### Handler
Handler机制是Android中常用的一种跨线程通讯的方式，一般的使用方法是new一个Handler对象并实现handlerMessage()方法，然后在需要使用的地方调用sendMessage()方法，那么Handler是怎么实现跨线程通讯的呢？
首先我们需要一个Handler对象，这个handler对象中包含一个Looper对象和一个MessageQueue对象，这个Looper对象就是当前线程的Looper，主线程有一个默认的Looper对象，子线程的话需要调用Looper.prepare()方法生成一个新的Looper对象，这里需要Looper在当前线程是唯一的，所以Looper构造函数是私有的，并且在该对象中有一个静态的ThreadLocal<Looper>来保存不同的线程的Looper对象保证唯一性，Looper的初始化的时候new了一个MessageQueue对象，该对象就是handler中的MessageQueue对象，在当前线程也是唯一的，这个时候Looper,MessageQueue，handler就联系在一起了，我们在看handle.sendMessage()，这里最终调用的是messageQueue中的enqueueMessage()方法，该方法就是把msg添加到MessageQueue指定的节点中，MessageQueue其实不是一个队列而是一个Message的单向链表，其中有个mMessage的参数指向链表的第一个节点，这个时候消息就已经成功保存到消息队列中了
接下来就是消息的读取了，前面有说到Handler中有一个Looper对象，Looper在创建成功之后还要调用loop()方法，该方法其实就是一个死循环一直查找MessageQueue的消息，然后调用msg.target.dispatchMessage(msg)将消息回调到handler,直到消息返回null退出循环，这里在MessageQueue里面查找消息调用的是MessageQueue的next()方法，该方法也是一个死循环，在这个里面使用了Linux里面一个特殊的阻塞机制，在阻塞的同时不会产生ANR

- **Looper：**
```java
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();//用于消息分发到相应的线程
private Looper(boolean quitAllowed) {//private不能通过构造函数创建
    mQueue = new MessageQueue(quitAllowed);  //每一个Looper对应一个MessageQueue对象
    mThread = Thread.currentThread();
}
```
```java
private static void prepare(boolean quitAllowed) {//创建Looper
        if (sThreadLocal.get() != null) {//每个线程只能有创建一个Looper
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
```java
public static void loop() {//轮询分发消息
        final Looper me = myLooper();//获取当前线程Looper
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;//拿到Looper对象中的MessageQueue对象
    	...
		for (;;) {
            if (msg == null) {
                // 调用的是MessageQueue的next（）方法，在MessageQueue的next方法中也是一个死循环，
                // 不过里面使用了nativePollOnce(ptr, nextPollTimeoutMillis)是Linux系统的一个阻塞方法，不会引起ANR
                Message msg = queue.next(); 
                // No message indicates that the message queue is quitting.
                return;
            }
            ...
            //这里是真正的分发过程，target为发送msg消息的handler对象，msg把自己当作参数传递给handler方法的dispatchMessage方法
			msg.target.dispatchMessage(msg);
            ...
        }

```

- Handler：
```java
final Looper mLooper;//当前线程的Looper对象
final MessageQueue mQueue;//Looper中的MessageQueue
public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {//最终通过这个方法发送消息
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
            long uptimeMillis) {
        msg.target = this;
        msg.workSourceUid = ThreadLocalWorkSource.getUid();

        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);//将消息转发到消息队列中
    }
```

- Message：
```java
Message next;//Message中有一个next的指针（说明Message对象是一个单向链表）
private static Message sPool;//静态对象指向已经回收的空Message，用于Message缓存
private static final int MAX_POOL_SIZE = 50;//最大缓存池大小
public void recycle(){...}//就是将各个参数重置为初始值，sPool重新指向这个对象实现复用
public static Message obtain() {//所以尽量使用obtain初始化Message
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```

- MessageQueue：
```java
Message mMessages;//消息队列的头指针，整个消息队列其实是一个单向链表
boolean enqueueMessage(Message msg, long when) {//
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();//消息回收
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {//立即处理这条消息
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {//将消息添加到队列的指定位置
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands(); //刷一下，就当是Android系统的一种性能优化操作
        }
        nativePollOnce(ptr, nextPollTimeoutMillis);//native底层实现堵塞，堵塞状态可被新消息唤醒,头一次进来不会延迟
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;//获取头节点消息
            if (msg != null && msg.target == null) {
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);//获取堵塞时间
                } else {
                    mBlocked = false;
                    if (prevMsg != null) { //头结点指向队列中第二个消息对象
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;   //直接出队列返回给looper
                }
            } else {
                nextPollTimeoutMillis = -1;//队列已无消息，一直堵塞
            }
            if (mQuitting) {
                dispose();
                return null;
            }
        pendingIdleHandlerCount = 0;
        nextPollTimeoutMillis = 0;
    }
}
    void quit(boolean safe) {
        if (!mQuitAllowed) {//主线程不允许退出
            throw new IllegalStateException("Main thread not allowed to quit.");
        }

        synchronized (this) {
            if (mQuitting) {
                return;
            }
            mQuitting = true;

            if (safe) {//Looper.quitSafely()移除所有延时消息
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();//Looper.quit()移除所有消息
            }

            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }
```
### Socket
### Pipe和signal
## 运行时态（Runtime）
### ART流程
### build过程->dex文件格式
### GC机制
### JNI
## 异常原理
### ANR
安卓ANR表示应用程序无法响应输入事件。一般activity 5s, Service 20s, BroadCastReceiver 10s.

### java crash和Native crash
### 卡顿

# 跨端方案
## Flutter


