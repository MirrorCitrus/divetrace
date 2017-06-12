# RxJava（1）：基础使用篇
## 概述
> RxJava is a Java VM implementation of ReactiveX (Reactive Extensions): a library for composing asynchronous and event-based programs by using observable sequences.

- RxJava是什么？

是一个Java实现的响应式编程库。

- RxJava的优点？

响应式的编程方式，简洁、清晰。也可以具体体现在如下3个方面：

1.解决异步嵌套问题
2.错误处理的嵌套问题
3.便于复杂的交互逻辑编写


## API介绍和原理解析

### RxJava的观察者模式
RxJava的观察者模式的组成：

- 被观察者：`Observable` 
- 观察者：`Observer` 
- 订阅：`subscribe()`
- 事件通知：`onNext()` , `onCompleted()`, `onError()`

### 基本实现
先看一个最简单的例子：这里将一个字符串数组中的内容依次添加“Hello”然后打印出来。

```
Observable.from(names).subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        Log.v(tag, "Hello " + s + "!");
    }
});
```

使用Lambda表达式，更加简洁：

```
Observable.from(names).subscribe((Action1) (s) -> {
        Log.v(TAG, "Hello " + s + "!");
    }
});
```

在这个例子中，我们做了三件事：

- 构建被观察者Observable。包括：指定被观察对象的类型、指定被订阅时要做什么。
- 构建观察者Observer。包括：指定观察对象的类型、指定观察的事件到来时要做什么。
- 完成“订阅”动作。

我们一步一步来看。

#### 1) 构建Observable

构建Observable需要做两件事：指定被观察对象的类型、指定被订阅时要做什么。前者通过Observable类自带的泛型参数实现；后者通过`Observable.OnSubscribe<T>`这个类的`call`方法，告诉Observer,当订阅来临时做什么。所以我们可以这么创建：

```
Observable.create(new Observable.OnSubscribe<String>() {

    @Override
    public void call(Subscriber<? super String> subscriber) {
        subscriber.onNext("world");
        subscriber.onCompleted();
    }
});
```

这段代码的意思是：一旦被订阅，我们就把一个String类型的对象“world”，通过onNext()方法发送给Observer，同时发送onCompleted方法，表示事件序列结束。可见这是Observable最简单最常用的行为，因此RxJava帮我们做了封装，我们可以用下面的代码来代替上面的代码：

```
Observable<String> observable = Observable.just("world");
```

如果我们要包装多个String，可以有如下两种写法：

```
Observable<String> observable = Observable.just("world", "haha");
Observable<String> observable = Observable.from(new String[] {"world", "haha"});
```

这两种写法是等价的，订阅者会依次收到`onNext("world")`, `onNext("haha")`, `onCompleted()`三个事件。注意到这里的返回值仍然是`Observable<String>`，说明我们观察的对象仍然是一个String对象，只不过它可以产生多个String事件。

另一方面，我们并没有主动调用onError方法，实际上，Observable已经帮助我们封装好了：如果`call`方法发生了什么异常，onError事件会自动被抛出来。

#### 2) 构建Observer

构建Observer需要做两件事：指定观察对象的类型、指定收到订阅的事件之后要做什么。我们需要这样创建：

```
Observer<String> observer = new Observer<String>() {
    @Override
    public void onNext(String s) {
        Log.d(TAG, "Hello " + s + "!");
    }

    @Override
    public void onCompleted() {
        Log.d(TAG, "Completed!");
    }

    @Override
    public void onError(Throwable e) {
        Log.d(TAG, "Error!");
    }
};
```

而通常，我们只关心onNext事件，所以这里我们可以使用一个快捷类`Action1`，来指定onNext的行为：

```
Action1 action = new Action1<String>() {
    @Override
    public void call(String s) {
        Log.v(TAG, "Hello " + s + "!");
    }
};

```
简洁版：

```
Action1 action = (Action1) (s) -> { Log.v(TAG, "Hello " + s + "!"); };
```

除了`Observer`接口之外，RxJava还内置了一个实现了`Observer`的抽象类：`Subscriber`。 它对Observer 接口进行了一些扩展，但他们的基本使用方式是完全一样的。实际上，RxJava最终还是会把我们创建的Observer对象转换成Subscriber对象来使用。那么Subscriber主要在Observer上扩展了什么呢？

- `onStart`:它会在subscribe刚开始，而事件还未发送之前被调用，可以用于做一些准备工作。
- `Subscription`：Subscriber同时实现了Subscription接口，提供取消订阅的方法。

#### 3) 完成订阅动作

超简单，调用Observable.subscribe方法即可。这个方法其实相当于做了如下三件事（精简的代码）:

```
public Subscription subscribe(Subscriber subscriber) {
    subscriber.onStart();
    onSubscribe.call(subscriber);
    return subscriber;
}
```

## 线程控制

### Scheduler的API

RxJava已经内置了几个Scheduler，它们已经适合大多数的使用场景：

- `Schedulers.immediate()`: 直接在当前线程运行，相当于不指定线程。这是默认的 Scheduler。
- `Schedulers.newThread()`: 总是启用新线程，并在新线程执行操作。
- `Schedulers.io()`:  I/O 操作（读写文件、读写数据库、网络信息交互等）所使用的 Scheduler。
- `Schedulers.computaion()`:计算所使用的 Scheduler。
- `AndroidSchedulers.mainThread()`: Android中的主线程。

有了这些调度器之后，我们就可以方便地指定线程了：

- `subscribeOn()`: 指定subscribe()方法调用后（被订阅时），被观察者发送事件的线程。即：`Observable.OnSubscribe<T>`的`call`方法执行的线程。
- `observeOn()`: 指定观察者收到事件后，处理的线程。即：onNext的线程

## 操作符

### 操作符一览

- 转换类操作符
    - `map`: 将一种类型的被观察对象，转换成另一种类型的被观察对象。即：事件对象的一对一转换。
    - `flatMap`: 将一种类型的被观察对象，转换成另一种类型的多个被观察对象。即：事件对象的一对多转换。
    - `concatMap`: （暂不做了解）
    - `switchMap`: （暂不做了解）
    - `scan`: （暂不做了解）
    - `groupBy`: （暂不做了解）
    - `lift`：转换操作符的实现基础：使得事件序列的处理和再发送成为可能。
- 过滤类操作符（超简单）
    - `filter(Func1)`: 过滤，参数`Func1<T, Boolean>`, 返回false表示舍弃
    - `take(int)`:
    - `takeLast(int)`:
    - `takeUntil(Func1)`:
    - `skip(int)`:
    - `skipLast(int)`:
    - `elementAt(int)`:
    - `debounce(long, TimeUnit)`:
    - `distinct()`:
    - `first()`:
    - `last()`:
- 组合类操作符
    - `merge(Observable, Observable)`: 将多个事件序列合并成一个（合并后无序）
    - `startWitch(T)`: 在一个事件序列前插入数据
    - `concat(Observable, Observable)`: 将两个事件序列合并成一个，合并后保证前一个事件序列结束后开始发送第二个事件
    - `zip(Observable, Observable, Func1)`: 将两个事件按照一定公式结合成一个新的被观察对象发出。
    - `combineLatest`: (暂不做了解)
    - `switchOnNext`: (暂不做了解)
    - `join`: (暂不做了解)
- 操作符组合与复用 
    - `compose`: 将针对一个Observable的多个变换封装起来，成为一个Transformer，后续使用compose方法来应用这个Transformer，目标：一次性应用多个变换、且该变换序列可复用。

### map和flatMap

将一种类型的被观察对象，转换成另一种类型的被观察对象。即：事件对象的一对一转换。map方法可以接收一个类型为`Func1<T, R>`的泛型类，指定如何将类型T转换为类型R，最终，Observer可以直接拿到一个R类型的对象来处理。下面是一个将路径转换为Bitmap，最终展示的例子：

```
Observable.just("images/logo.png") // 输入类型 String
    .map(new Func1<String, Bitmap>() {
        @Override
        public Bitmap call(String filePath) { // 参数类型 String
            return getBitmapFromPath(filePath); // 返回类型 Bitmap
        }
    })
    .subscribe(new Action1<Bitmap>() {
        @Override
        public void call(Bitmap bitmap) { // 参数类型 Bitmap
            showBitmap(bitmap);
        }
    });
```

和map相对应，flatMap要解决的是事件对象的一对多转换的问题。举例，我们有个Student类型的数据结构，Student有一个`courses:List<Course>`这样的成员变量，如果我们要将每一门课程和学生的对应关系依次打印出来，最优雅的做法是什么呢？

如果说，我们把每个Student作为一个被观察者，我们期望每个这样的事件可以转化为多个Course事件发出来。回顾我们之前提到的`Observable.from()`方法，我们可以有一种`Observable<R>`对象，它会发送多个`onNext(R)`事件后才结束。那么这个问题就可以变成：我们需要一种映射，将T转换为一个可以发送多个事件的`Observable<R>`。

这就是flatMap，取名为“平铺”，也是比较形象的。下面是一个flatMap的例子：

```
Observable.from(students) // 输入类型：Student[]
    .flatMap(new Func1<Student, Observable<Course>>() {
        @Override
        public Observable<Course> call(Student student) { // 每个Student转换成一个Observable<Course>
            return Observable.from(student.getCourses());
        }
    })
    .subscribe(new Subscriber<Course>() {
        @Override
        public void onNext(Course course) {
            Log.d(tag, course.getName()); // 响应Course事件
        }
    
    });
```


### 操作符组合compose

`compose`操作符是对多个操作符的组合，以便一系列的转换可以封装器来复用。具体来讲，我们将一个被观察对象`Observable<T>`经过一系列操作符转换后，形成一个`Observable<R>`，我们将这一系列的操作符封装成一个`Transformer<T, R>`,实现它的`call(Observable<T>):Observable<R>`方法。后续，使用`compose`方法直接应用这个Transformer即可。举例：

```
public class LiftAllTransformer implements Observable.Transformer<Integer, String> {
    @Override
    public Observable<String> call(Observable<Integer> observable) {
        return observable
            .lift1()
            .lift2()
            .lift3()
            .lift4();
    }
}

Transformer liftAll = new LiftAllTransformer();
observable1.compose(liftAll).subscribe(subscriber1);
observable2.compose(liftAll).subscribe(subscriber2);
observable3.compose(liftAll).subscribe(subscriber3);
observable4.compose(liftAll).subscribe(subscriber4);
```

## References

- [ ] [Awesome-RxJava GitHub：RxJava学习资料总结](https://github.com/lzyzsd/Awesome-RxJava)
- [x] [给Android开发者的RxJava详解](https://gank.io/post/560e15be2dca930e00da1083#toc_3)