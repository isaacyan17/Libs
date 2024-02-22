# Rxjava

## 背景

![](https://cdn.nlark.com/yuque/0/2021/jpeg/1227097/1637724976002-59a8aa4c-db69-4615-911c-f3c94e23941e.jpeg)
## 创建一个订阅流程

```java

Observable.create(new ObservableOnSubscribe<String>() {
    //创建一个被观察者,定义事件
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {

            }
        })
    //new Observer: 创建一个观察者,处理事件
    //subscribe: 订阅者,关联被观察者和观察者,分发事件
    .subscribe(new Observer<String>() {
            @Override
            public void onSubscribe( Disposable d) {
                
            }

            @Override
            public void onNext( String s) {

            }

            @Override
            public void onError( Throwable e) {

            }

            @Override
            public void onComplete() {

            }
        });
```
我们应该熟悉这样的定义:

| 类型 | 定义 | 作用 |
| --- | --- | --- |
| Observable | 被观察者 | 产生事件 |
| subscribe | 订阅者 | 连接观察者和被观察者 |
| Observer | 观察者 | 处理事件 |

## 订阅关系

观察者和被观察者是如何被关联的?

步骤1：创建被观察者(Observable),  定义需发送的事件
步骤2：创建观察者(Observer) ,  定义响应事件的行为
步骤3：通过订阅（subscribe）连接观察者和被观察者

```java
 //分析1
Observable.create(new ObservableOnSubscribe<String>() {
    //创建一个被观察者,定义事件
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {

            }
        })
    
    
/*
 源码 Observable.create
*/
@SchedulerSupport(SchedulerSupport.NONE)
    //分析1:ObservableOnSubscribe
    public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
   		//分析2: new ObservableCreate
    	//分析3: RxJavaPlugins.onAssembly
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }

...
/*
	源码
	分析1, ObservableOnSubscribe是一个接口 
    代码中我们会手动创建一个ObservableOnSubscribe的实现对象
*/
public interface ObservableOnSubscribe<T> {

    /**
     * Called for each Observer that subscribes.
     * @param e the safe emitter instance, never null
     * @throws Exception on error
     */
    void subscribe(@NonNull ObservableEmitter<T> e) throws Exception;
}


/*
源码
	分析2,ObservableCreate
*/
final ObservableOnSubscribe<T> source;

//接收一个ObservableOnSubscribe的数据,保存起来
 public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
 }


//ObservableCreate继承了Observerble,实现抽象方法subscribeActual
//这个方法并没有在create时执行,但是到后面subscribe以后,observer会陆续
//触发ObserverbleCreate中定义的方法.

  protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);

        try {
            //source就是观察者（Observable）中创建的ObservableOnSubscribe对象
            
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }

/*
 源码
 分析3: RxJavaPlugins.onAssembly
*/
 public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
        Function<? super Observable, ? extends Observable> f = onObservableAssembly;
        if (f != null) {
            return apply(f, source);
        }
     //返回的source即ObserverbleCreate
        return source;
    }
//总结,create方法中返回了ObserverbleCreate,并且其中已经创建好了OBserver,即观察者的回调方法.
//这些方法都是关系订阅以后,会执行的流程.




```
```java
subscribe(Observer<? super T> observer)
//分析订阅方法
// 
    
    @Override
    public final void subscribe(Observer<? super T> observer) {
       ...
           //this就是Observerble,利用RxJavaPlugins将观察者和被观察者关联.
            observer = RxJavaPlugins.onSubscribe(this, observer);
    //执行subscribeActual方法,这个方法是一个抽象方法,它最终会被ObserverbleCreate实现类执行.
            subscribeActual(observer);
    ...

    }
```

## 多个Observerble的事件发送顺序

比如如下情况
```java
 // 创建第1个被观察者（Observable1）
    Observable.create(new ObservableOnSubscribe<Integer>() {
                @Override
                public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                    emitter.onNext(1);
                    emitter.onNext(2);
                    emitter.onNext(3);
                }
            })
            // 使用flatMap操作符（内部会创建第2个被观察者（Observable2））
            .flatMap(new Function<Integer, ObservableSource<String>>() {
                @Override
                public ObservableSource<String> apply(Integer integer) throws Exception {
                    final List<String> list = new ArrayList<>();
                    for (int i = 0; i < 3; i++) {
                        list.add("我是事件" + integer + "拆分后的子事件" + i);
                        // 通过flatMap中将被观察者生产的事件序列先进行拆分，再将每个事件转换为一个新的发送三个String事件
                        // 最终合并，再发送给被观察者
                    }
                    return Observable.fromIterable(list);
                }

            })
            .subscribe(new Observer<String>() {
                @Override
                public void onSubscribe(Disposable d) {
                    Log.d(TAG, "开始采用subscribe连接");
                }
                // 默认最先调用复写的 onSubscribe（）

                @Override
                public void onNext(String value) {
                    Log.d(TAG, "响应事件："+ value   );
                }

                @Override
                public void onError(Throwable e) {
                    Log.d(TAG, "对Error事件作出响应");
                }

                @Override
                public void onComplete() {
                    Log.d(TAG, "对Complete事件作出响应");
                }
            });
```
它会先调用Observerble2的`subscribe(Observer)`, `subscribeActual(Observer)`, 然后回调Observerble1的`subscribe(Observer)`, `subscribeActual(Observer)`. 所以是顺序执行的
