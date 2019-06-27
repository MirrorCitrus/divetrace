> 共4篇
* [RxJava(1): 基础使用篇](RxJava1.md)
* [RxJava(2): 原理篇](RxJava2.md)
* [RxJava(3): RxJava 2.x和RxJava 1.x的区别](RxJava3.md)
* [RxJava(4): 应用篇](RxJava4.md)




# RxJava(3)-RxJava2.x和RxJava1.x的区别

## Maven地址和依赖包的更改
为了让 RxJava 1.x 和 RxJava 2.x 相互独立，我们把RxJava 2.x 被放在了maven `io.reactivex.rxjava2:rxjava:2.x.y` 下，类放在了`io.reactivex`包下。

```
// 原本：`RxJava 1.0` 依赖
compile 'io.reactivex:rxandroid:1.2.0'
compile 'io.reactivex:rxjava:1.1.5'

// 更改：`RxJava 2.0` 依赖
compile 'io.reactivex.rxjava2:rxandroid:2.0.1'
compile 'io.reactivex.rxjava2:rxjava:2.0.7'
```


## 基础流程的变化

下面是RxJava 2.x里的基础示例：

```
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                e.onNext(1);
                e.onNext(2);
                e.onNext(3);
                e.onComplete();
            }
        }).subscribe(new Observer<Integer>() {

            private int i;
            private Disposable mDisposable;

            @Override
            public void onSubscribe(@NonNull Disposable d) {      
                mDisposable = d;
            }

            @Override
            public void onNext(@NonNull Integer integer) {
                i++;
                if (i == 2) {
                    // 在RxJava 2.x 中，新增的Disposable可以做到切断的操作，让Observer观察者不再接收上游事件
                    mDisposable.dispose();
                }
            }

            @Override
            public void onError(@NonNull Throwable e) {
                Log.e(TAG, "onError : value : " + e.getMessage() + "\n" );
            }

            @Override
            public void onComplete() {
                Log.e(TAG, "onComplete" + "\n" );
            }
        });
```

### Observable构建方式的变化

- 创建Observable的参数发生了变化，从`OnSubscribe`变成了`ObservableOnSubscribe`
- 构建`ObservableOnSubscribe`参数时，需要重写的方法也发生了变化，原先是`call(Subscriber)`，现在是`subscribe(ObservableEmitter)`。

### Observer构建方式的变化
其次是创建Observer的方法发生了变化，需要实现一个onSubscribe(@NonNull Disposable d)接口，其中，新增的Disposable参数可以做到切断的操作，让Observer观察者不再接收上游事件

RxJava 1.x构建Observer有两种方式，一种是构建`Observer`对象，一种是构建`Subscriber`对象。
RxJava 2.x则需要构建`Observer`对象，`Observer`类的改动包括两点：
- 增加回调方法`onSubscribe(Disposable)`，作用：最先调用该方法，即适合做初始化工作。其中的Disposable参数作用类似于Subsciption，用于切断观察者和被观察者的订阅关系
- onCompleted()方法改名为onComplete()

### 订阅方式的变化

- ActionX发生了变化

2.x中没有定义Action3-Action9和ActionN。Action0 被操作符Action和Scheduler代替。Action1被重命名为Consumer。Action2 被重命名为BiConsumer。 ActionN被Consumer代替。

- FuncX发生了变化

定义了Function和BiFunction, 把Func3 - Func9 分别改成了 Function3 - Function9 。FuncN被Function代替。

- RxJava2的接口方法都允许抛出异常

Action#run()方法，Consumer#accept()方法，都允许抛出异常，这意味着，在这些方法里调用会发生异常的方法不需要try-catch。

- subscribe()方法的返回值变成Disposable

subscribe()方法的返回值不再是`Subscription`，而是变成`Disposable`。

## 基本数据源：Observable, Flowable, Completable, Single, Maybe

- `Flowable`: 0..N flows, 支持背压
- `Observable`: 0..N flows, 不支持背压
- `Single`: 只发射一条单一的数据，或者一条异常通知，不能发射完成通知，其中数据与异常通知只能发射一个
- `Completable`: 只发射一条完成通知，或者一条异常通知，不能发射数据，其中完成通知与异常通知只能发射一个
- `Maybe`: 不发射数据(此时会发送onComplete)，或发射一条单一的数据(不发送onComplete)，或者一条异常通知，其中完成通知和异常通知只能发射一个，发射数据只能在发射完成通知或者异常通知之前，否则发射数据无效

### Flowable和背压

#### 背压（backpressure）
当上下游在不同的线程中，通过Observable发射，处理，响应数据流时，如果上游发射数据的速度快于下游接收处理数据的速度，这样对于那些没来得及处理的数据就会造成积压，这些数据既不会丢失，也不会被垃圾回收机制回收，而是存放在一个异步缓存池中，如果缓存池中的数据一直得不到处理，越积越多，最后就会造成内存溢出，这便是响应式编程中的背压（backpressure）问题。

#### Flowable——解决背压问题

在Rxjava2中，Flowable可以看做是为了解决背压问题，在Observable的基础上优化后的产物，与Observable不处在同一组观察者模式下，Observable是ObservableSource/Observer这一组观察者模式中ObservableSource的典型实现，而Flowable是Publisher与Subscriber这一组观察者模式中Publisher的典型实现。

由于Flowable处理了背压问题，因此性能比Observable慢的多。因此，只有在需要处理背压问题时，才需要使用Flowable。更详细来说，使用Flowable必须同时满足如下几点，否则应使用Observable:

- 上下游运行在不同线程中
- 下游处理数据的速度慢于上游发射数据的速度
- 数据流中不只有一条数据

Flowable的示例：
```
Flowable
    .create(new FlowableOnSubscribe<Integer>() {
        @Override
        public void subscribe(FlowableEmitter<Integer> e) throws Exception {
            System.out.println("发射----> 1");
            e.onNext(1);
            System.out.println("发射----> 2");
            e.onNext(2);
            System.out.println("发射----> 3");
            e.onNext(3);
            System.out.println("发射----> 完成");
            e.onComplete();
        }
    }, BackpressureStrategy.BUFFER) //create方法中多了一个BackpressureStrategy类型的参数
    .subscribeOn(Schedulers.newThread()) //为上下游分别指定各自的线程
    .observeOn(Schedulers.newThread())
    .subscribe(new Subscriber<Integer>() {
        @Override
        public void onSubscribe(Subscription s) {   //onSubscribe回调的参数不是Disposable而是Subscription
            s.request(Long.MAX_VALUE); // 设置下游请求数量，实现响应式拉取
        }

        @Override
        public void onNext(Integer integer) {
            System.out.println("接收----> " + integer);
        }

        @Override
        public void onError(Throwable t) {
        }

        @Override
        public void onComplete() {
            System.out.println("接收----> 完成");
        }
    });
```

#### BackpressureStrategy背压策略

当上游发送数据的速度快于下游接收数据的速度，且运行在不同的线程中时，Flowable通过自身特有的异步缓存池，来缓存没来得及处理的数据，缓存池的容量上限为128。BackpressureStrategy的作用便是用来设置Flowable通过异步缓存池缓存数据的策略，是五个枚举值。

- ERROR

对应ErrorAsyncEmitter。在此策略下，如果放入Flowable的异步缓存池中的数据超限了，则会抛出MissingBackpressureException异常。

- DROP

对应DropAsyncEmitter。在此策略下，如果Flowable的异步缓存池满了，会丢掉上游发送的数据。

- LATEST

对应LatestAsyncEmitter。与Drop策略一样，如果缓存池满了，会丢掉将要放入缓存池中的数据，不同的是，不管缓存池的状态如何，LATEST都会将最后一条数据强行放入缓存池中，来保证观察者在接收到完成通知之前，能够接收到Flowable最新发射的一条数据。

- BUFFER

对应BufferAsyncEmitter。此策略下，如果Flowable默认的异步缓存池满了，会通过此缓存池暂存数据，它与Observable的异步缓存池一样，可以无限制向里添加数据，不会抛出MissingBackpressureException异常，但会导致OOM。不同的是使用Flowable内存增长的速度要慢得多，那是因为基于Flowable发射的数据流，以及对数据加工处理的各操作符都添加了背压支持，附加了额外的逻辑。

- MISSING

对应MissingEmitter。在此策略下，通过Create方法创建的Flowable相当于没有指定背压策略，不会对通过onNext发射的数据做缓存或丢弃处理，需要下游通过背压操作符（onBackpressureBuffer()/onBackpressureDrop()/onBackpressureLatest()）指定背压策略。

#### Subscription和响应式拉取

RxJava 1.x的`Subscription`被更名为`Disposable`，它的`dispose()`方法用于解除Observer和Observable的订阅关系。

RxJava 2.x的`Subscription`则是一个新的类，`Subscription.cancel()`用于取消Subscriber与Flowable的订阅关系。

同时，新的`Subscription`多了一个方法`request(long n)`，它是用来设置下游对数据的请求数量，上游可以根据下游的需求量，按需发送数据，从而实现Flowable的响应式拉取。当下游通过这个方法设置了请求数量后，上游的`FlowableEmitter`通过`FlowableEmitter.requested()`方法来查询下游的需求量，实现按需发送。


### Single

```
Single.create(new SingleOnSubscribe<Integer>() {
        @Override
        public void subscribe(SingleEmitter<Integer> singleEmitter) throws Exception {
            singleEmitter.onSuccess(0); // 发送item
            // singleEmitter.onError(new RuntimeException("oops")); 或者发送一个异常
        }
    }).subscribe(new SingleObserver<Integer>() {
        @Override
        public void onSubscribe(Disposable disposable) {
            
        }

        @Override
        public void onSuccess(Integer integer) {

        }

        @Override
        public void onError(Throwable throwable) {

        }
    });
```

可以看到，和Single搭配使用的类还包括：`SingleOnSubscribe`, `SingleObserver`, `SingleEmitter`, 其中，`SingleEmitter`定义了`onSuccess(T)`和`onError(Throwable)`两个方法，两者只能选其一调用一次。

### Completable
和Completable搭配使用的类还包括：`CompletableOnSubscribe`, `CompletableObserver`, `CompletableEmitter`, 其中，`CompletableEmitter`定义了`onComplete()`和`onError(Throwable)`两个方法，两者只能选其一调用一次。

### Maybe
和Maybe搭配使用的类还包括：`MaybeOnSubscribe`, `MaybeObserver`, `MaybeEmitter`, 其中，`MaybeEmitter`定义了`onComplete()`, `onSuccess(T)`和`onError(Throwable)`方法，三者只能选其一调用一次。

## 其他区别

### Nulls
null不再可以作为Observable源来发送，Observable<Void>也不再可用，即：类似下面的操作会抛出NullPointerException：

```
Observable.just(null);

Single.just(null);

Observable.fromCallable(() -> null)
    .subscribe(System.out::println, Throwable::printStackTrace);

Observable.just(1).map(v -> null)
    .subscribe(System.out::println, Throwable::printStackTrace);
```

如果原来是通过null构建了一个作为信号的Observable，现在可以考虑改成一个共用的Object或enum，如：
```
num Irrelevant { INSTANCE; }

Observable<Object> source = Observable.create((ObservableEmitter<Object> emitter) -> {
   emitter.onNext(Irrelevant.INSTANCE);
   ...
});

```

### Subjects 和 Processors
随着Observable/Flowable的分离，XxxSubject不支持背压，支持背压的类都是FlowableProcessor的子类。

即：在2.x中，`AsyncSubject`, `BehaviorSubject`, `PublishSubject`, `ReplaySubject`和`UnicastSubject`不支持背压。

`AsyncProcessor`, `BehaviorProcessor`, `PublishProcessor`, `ReplayProcessor`和`UnicastProcessor`支持背压。

### Reactive-Streams compliance

### Runtime Hooks

### Schedulers的改变

### reactive-stream的创建和使用上的变化

### test

### 操作符




## References
- [x] [这可能是最好的RxJava 2.x 教程（完结版）](https://www.jianshu.com/p/0cd258eecf60)
- [ ] [What's different in 2.0](https://github.com/ReactiveX/RxJava/wiki/What's-different-in-2.0#nulls)
- [x] [Android ：RxJava2.0到底更新了什么？（含使用建议）](http://www.androidchina.net/8167.html)
- [x] [Rxjava2入门教程六：Single、Completable、Maybe——简化版的Observable](https://www.jianshu.com/p/66a55abbadef)
- [x] [Rxjava2入门教程五：Flowable背压支持——对Flowable最全面而详细的讲解](https://www.jianshu.com/p/ff8167c1d191)
- [ ] [Android RxJava ：图文详解 背压策略](https://www.jianshu.com/p/ceb48ed8719d)