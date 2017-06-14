# RxJava(2): 原理分析

> 代码基于：rxjava:1.3.0

## 基本流程 

// TODO 

## 转换类操作符的实现

### map

先看一个map的示例：

```
Observable.just(1)
        .map(new Func1<Integer, String>() {
            @Override
            public String call(Integer integer) {
                return "This is " + integer;
            }
        })
        .subscribe(new Action1<String>() {
            @Override
            public void call(String s) {
                Log.v(TAG, s);
            }
        });
```

这个示例中，我们首先构建了一个`Observable<Integer>`，然后通过`map`方法将其转换为`Observable<String>`。map方法接收一个function对象指定如何进行转换：`Func<Integer, String>`。

我们现在抽象一下类型：我们希望使用map实现类型T到R的转换，那么map的实现是这样的：首先，构建一个`OnSubscribeMap<T, R>`对象，然后由这个对象构建一个新的被观察对象：`Observable<R>`后返回。

```
public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
    return unsafeCreate(new OnSubscribeMap<T, R>(this, func));
}
```

OnSubscriveMap的代码是这样的：

```
public final class OnSubscribeMap<T, R> implements OnSubscribe<R> {

    final Observable<T> source;

    final Func1<? super T, ? extends R> transformer;

    public OnSubscribeMap(Observable<T> source, Func1<? super T, ? extends R> transformer) {
        this.source = source;
        this.transformer = transformer;
    }

    @Override
    public void call(final Subscriber<? super R> o) {
        MapSubscriber<T, R> parent = new MapSubscriber<T, R>(o, transformer);
        o.add(parent);
        source.unsafeSubscribe(parent);
    }
}
```

OnSubscriveMap实现了`OnSubscribe<R>`接口，它拦截了最终的`Subscriber<R>`对象对原始的`Observable<T>`的订阅动作，由自己响应。响应的内容是构建一个新的`Subscriber<T>`对象，并订阅到原始的`Observable<T>`。这个新的`Subscriber<T>`对象会在`onNext(T)`的响应中，先使用之前的入参`Func1<T, R>`把T类型的对象转换成R类型，然后调用最终的subscriber.onNext方法，将转换后的被观察对方发送出去。

这样，就很好理解`MapSubscriber<T, R>`的作用：它订阅了原始的被观察事件，然后将被观察对象进行转换后，发送给最终的观察者。下面是精简过的代码：

```
static final class MapSubscriber<T, R> extends Subscriber<T> {

    final Subscriber<? super R> actual;

    final Func1<? super T, ? extends R> mapper;

    public MapSubscriber(Subscriber<? super R> actual, Func1<? super T, ? extends R> mapper) {
        this.actual = actual;
        this.mapper = mapper;
    }

    @Override
    public void onNext(T t) {
        R result;

        try {
            result = mapper.call(t);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            unsubscribe();
            onError(OnErrorThrowable.addValueAsLastCause(ex, t));
            return;
        }

        actual.onNext(result);
    }

    @Override
    public void onError(Throwable e) {
        actual.onError(e);
    }


    @Override
    public void onCompleted() {
        actual.onCompleted();
    }
}

```

其实类似于拦截和代理，最后我们再回顾和梳理一下整个map的过程：

- `Subscriber<R>`对象开始订阅
- 新建`OnSubscribeMap<T, R>`对象(继承`OnSubscribe<R>`)拦截订阅事件，再触发一次对原始Observable的订阅
    - 注：新建的`OnSubscribe<R>`用来构建了新的`Observable<R>`。
- 新建`MapSubscriber<T, R>`对象(继承`Subscriber<T>`)拦截原始onNext事件，将事件对象转换后，传递给实际的subscriber
- 实际的`Subscriber<R>`对象收到onNext事件



## 线程调度的实现

### subscribeOn()的实现

```
public final Observable<T> subscribeOn(Scheduler scheduler, boolean requestOn) {
    // ...
    return unsafeCreate(new OperatorSubscribeOn<T>(this, scheduler, requestOn));
}
```

subscribeOn方法本身也属于一种操作符，所以和前面的map操作很类似。这里构建了一个`OperatorSubscribeOn`对象，然后再构建一个新的`Observable<T>`对象返回。我们重点关注`OperatorSubscribeOn`的作用。

```
public final class OperatorSubscribeOn<T> implements OnSubscribe<T> {

    final Scheduler scheduler;
    final Observable<T> source;
    final boolean requestOn;

    public OperatorSubscribeOn(Observable<T> source, Scheduler scheduler, boolean requestOn) {
        this.scheduler = scheduler;
        this.source = source;
        this.requestOn = requestOn;
    }

    @Override
    public void call(final Subscriber<? super T> subscriber) {
        final Worker inner = scheduler.createWorker();

        SubscribeOnSubscriber<T> parent = new SubscribeOnSubscriber<T>(subscriber, requestOn, inner, source);
        subscriber.add(parent);
        subscriber.add(inner);

        inner.schedule(parent);
    }
}
```

同样地，它也实现了`OnSubscribe<T>`接口，同样是拦截了原始订阅消息，并构建了一个新的Subscriber，即`SubscribeOnSubscriber<T>`。但是它没有直接触发对Observable的订阅，而是使用`Scheduler.createWorker().schedule(Action0)`来将一个动作传递到指定线程去执行，也就是说，这里发生了线程调度。

可想而知，这里所说的动作，就是“对Observable的订阅、对原始onNext的拦截和转发”。在`SubscribeOnSubscriber<T>`类的实现中，比较清晰明了，只贴出它作为`Action0`的call方法：

```
static final class SubscribeOnSubscriber<T> extends Subscriber<T> implements Action0 {

    final Subscriber<? super T> actual;

    final boolean requestOn;

    final Worker worker;

    Observable<T> source;

    Thread t;

    @Override
    public void call() {
        Observable<T> src = source;
        source = null;
        t = Thread.currentThread();
        src.unsafeSubscribe(this);
    }
}
```

### observeOn()的实现

同样，直接看代码：

```
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    // ...
    return lift(new OperatorObserveOn<T>(scheduler, delayError, bufferSize));
}

public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
    return unsafeCreate(new OnSubscribeLift<T, R>(onSubscribe, operator));
}
```

observeOn()是借助于lift来实现的，相比subscribeOn()相对复杂些。不过最终还是在`lift`方法中创建了一个新的Observable。

我们先看lift的原理。lift是很多变换类操作符的基础，它接收一个Operator作为参数，可以对事件进行拦截、处理和再发送。和前面的分析流程很像，这里涉及到新建Observable和新建Subscriber的操作。先看看创建最终的Observable时，其构造参数`OnSubscribeLift<T, R>`做了什么：

```
public final class OnSubscribeLift<T, R> implements OnSubscribe<R> {

    final OnSubscribe<T> parent;

    final Operator<? extends R, ? super T> operator;

    public OnSubscribeLift(OnSubscribe<T> parent, Operator<? extends R, ? super T> operator) {
        this.parent = parent;
        this.operator = operator;
    }

    @Override
    public void call(Subscriber<? super R> o) {
        Subscriber<? super T> st = RxJavaHooks.onObservableLift(operator).call(o);
        st.onStart();
        parent.call(st);
    }
}
```

和前面的流程基本一致：新建的`OnSubscribeLift<T, R>`，继承于`OnSubscribe<R>`，拦截了最终的Subscriber的订阅行为，在call方法中，先对subscriber应用一次操作符生成一个新的subscriber，再使用新的subscriber向原始的被观察对象发起订阅。

按照这样分析，我们也很好理解“操作符”的概念：`Operator<R, T>`, 是能够将`Subscriber<R>`转换为`Subscriber<T>`的类：

```
public interface Operator<R, T> extends Func1<Subscriber<? super R>, Subscriber<? super T>> {
    // 方法定义在Func1中，这里只是写出来进行示意
    Subscriber<? super T> call(Subscriber<? super R> sub);
}
```

回到最初的observeOn方法，我们传递了一个`OperatorObserveOn<T>`对象作为操作符：

```
public final class OperatorObserveOn<T> implements Operator<T, T> {

    private final Scheduler scheduler;
    private final boolean delayError;
    private final int bufferSize;

    @Override
    public Subscriber<? super T> call(Subscriber<? super T> child) {
        if (scheduler instanceof ImmediateScheduler) {
            // avoid overhead, execute directly
            return child;
        } else if (scheduler instanceof TrampolineScheduler) {
            // avoid overhead, execute directly
            return child;
        } else {
            ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError, bufferSize);
            parent.init();
            return parent;
        }
    }

}
```

步骤比较简单，创建了一个新的Subscriber对象：`ObserveOnSubscriber`，并调用它的init方法。

```
static final class ObserveOnSubscriber<T> extends Subscriber<T> implements Action0 {
    final Subscriber<? super T> child;
    final Scheduler.Worker recursiveScheduler;

    public ObserveOnSubscriber(Scheduler scheduler, Subscriber<? super T> child, boolean delayError, int bufferSize) {
        this.child = child;
        this.recursiveScheduler = scheduler.createWorker();
        // ... 
    }

    void init() {
        Subscriber<? super T> localChild = child;
        localChild.setProducer(new Producer() { 
            // ... 
        });
        localChild.add(recursiveScheduler);
        localChild.add(this);
    }

    @Override
    public void onNext(final T t) {
        // ...
        schedule();
    }

    @Override
    public void onCompleted() {
        // ...
        schedule();
    }

    @Override
    public void onError(final Throwable e) {
        // ...
        schedule();
    }

    protected void schedule() {
        if (counter.getAndIncrement() == 0) {
            recursiveScheduler.schedule(this);
        }
    }

    // only execute this from schedule()
    @Override
    public void call() {
        // ...
        localChild.onNext(NotificationLite.<T>getValue(v));
        
        // 这里面的逻辑比较复杂，有其他的判断分支用于调用onError和onCompleted，这里仅示意一下，不再贴了
    }

}
```
最后，我们回顾并总结一下`subscribeOn`和`observeOn`：

- `subscribeOn()`: 指定subscribe()方法调用后（被订阅时），被观察者发送事件的线程。即：`Observable.OnSubscribe<T>`的`call`方法执行的线程。
- `observeOn()`: 指定观察者收到事件后，处理的线程。即：`onNext`的线程

### 线程调度

由前面的分析可知，线程调度其实就是靠`Scheduler.createWorker().schedule(Action0)`来实现的。我们先看下`Scheduler`的结构：

```
public abstract class Scheduler {
    
    public abstract Worker createWorker();
    
    public abstract static class Worker implements Subscription {
        
        public abstract Subscription schedule(Action0 action);
            
    }
}
```

结构很简单，Scheduler最重要的方法就是createWorker()，创建一个Worker对象；Worker对象有一个schedule(Action0)方法，执行真正的线程调度。

// TODO

#### Schedulers.io()

#### Schedulers.newThread()

#### Schedulers.computation()

#### AndroidSchedulers.mainThread()


## References

- [x] [RxJava系列6(从微观角度解读RxJava源码)](https://zhuanlan.zhihu.com/p/22338235)