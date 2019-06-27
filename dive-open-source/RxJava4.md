> 共4篇
* [RxJava(1): 基础使用篇](RxJava1.md)
* [RxJava(2): 原理篇](RxJava2.md)
* [RxJava(3): RxJava 2.x和RxJava 1.x的区别](RxJava3.md)
* [RxJava(4): 应用篇](RxJava4.md)

# RxJava(4)-应用篇

## 典型示例

### 使用Schedulers执行后台任务

```
Observable.just(true)
        .map(aBoolean -> {
            _doSomeLongOperation_thatBlocksCurrentThread();
            return aBoolean;
        })
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(aBoolean -> System.out.print("onNext with return value: " + aBoolean));
```

### 收集连续事件（使用buffer）

```
RxView.clickEvents(_tapBtn)
        .map(onClickEvent -> {
            _log("GOT A TAP");
            return 1;
        })
        .buffer(2, TimeUnit.SECONDS)
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Action1<List<Integer>>() {
            @Override
            public void call(List<Integer> integers) {
                _log(String.format("%d taps", integers.size()));
            }
        });
```

### 搜索text监听（使用debounce）

```
RxTextView.textChangeEvents(_inputSearchText)
              .debounce(400, TimeUnit.MILLISECONDS)// default Scheduler is Computation
              .filter(changes -> isNotNullOrEmpty(_inputSearchText.getText().toString()))
              .observeOn(AndroidSchedulers.mainThread())
              .subscribe(onTextChangeEvent -> _log("Searching for " + onTextChangeEvent.text().toString()));
```

### 使用zip

zip操作符的作用：将两个Observable发射的事件按照一定公式合并成一个新的Observable发出。

```
// 根据用户名和repo名称，先查询所有的contributors列表，然后查询每个contributor的详细信息，最后将contributors详细信息打印出来
_githubService.contributors(_username.getText().toString(), _repo.getText().toString())
        .flatMap(Observable::from)
        .flatMap(contributor -> {
            Observable<User> _userObservable = _githubService.user(contributor.login)
                    .filter(user -> !isEmpty(user.name) && !isEmpty(user.email));

            return Observable.zip(_userObservable,
                    Observable.just(contributor),
                    Pair::new);
        })
        .subscribeOn(Schedulers.newThread())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(pair -> {
                    User user = ((Pair<User, Contributor>) pair).first;
                    Contributor contributor = ((Pair<User, Contributor>) pair).second;

                    _adapter.add(format("%s(%s) has made %d contributions to %s",
                            user.name,
                            user.email,
                            contributor.contributions,
                            _repo.getText().toString()));

                    _adapter.notifyDataSetChanged();
                }
        );

// 数据来源不同，部分来源于本地，部分来源于网络，进行合并显示时
Observable.zip(
        queryContactsFromLocation(),
        queryContactsForNet(),
        (Func2<List<Contacter>, List<Contacter>, List<Contacter>>) (contacters, contacters2) -> {
            contacters.addAll(contacters2);
            return contacters;
        })
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(contacters -> initPage(contacters));
```

### 使用PublishSubject发送UI事件
PublishSubject操作符：与普通的Subject不同，在订阅时并不立即触发订阅事件，而是允许我们在任意时刻手动调用onNext(),onError(),onCompleted来触发事件。

由此可以先定义PublishSubject和subscribe时的响应动作，再在UI事件（如onClick, onTextChange）发生时调用onNext，触发事件序列的发送。

```
public class DoubleBindingTextViewFragment extends BaseFragment {
    PublishSubject<Float> _resultEmitterSubject;

    @Override
    public View onCreateView(...) {
        ...
        _resultEmitterSubject = PublishSubject.create();
        _subscription = _resultEmitterSubject
              .asObservable()
              .subscribe(aFloat -> _result.setText(String.valueOf(aFloat)));
        ...
    }

    @OnTextChanged({R.id.double_binding_num1, R.id.double_binding_num2})
    public void onNumberChanged() {
        float num1 = Float.parseFloat(_number1.getText().toString());
        float num2 = Float.parseFloat(_number2.getText().toString());

        _resultEmitterSubject.onNext(num1 + num2);
    }
}
```

### 轮询（使用interval、repeatWhen）

```
// 简单轮询：每隔一个固定间隔时间轮询一次
Observable.interval(INITIAL_DELAY, POLLING_INTERVAL, TimeUnit.MILLISECONDS)
        .map(this::_doNetworkCallAndGetStringResult)
        .take(pollCount)
        .subscribe(taskName -> _log("Executing polled task [%s] now time : [xx:%02d]" + taskName));

// 复杂轮询：使用repeatWhen实现可变延迟策略的轮询
Observable.just(1)
    .repeatWhen(new Func1<Observable<? extends Void>, Observable<?>>() {
        @Override
        public Observable<?> call(Observable<? extends Void> completed) {

            return completed.flatMap(new Func1<Void, Observable<?>>() {
                static final int REPEAT_LIMIT = 3;
                int _repeatCount = 1;

                @Override
                public Observable<?> call(Void blah) {

                    if (_repeatCount >= REPEAT_LIMIT) {
                        return Observable.empty(); // terminate the sequence cause we reached the limit
                    }

                    // since we don't get an input
                    // we store state in this handler to tell us the point of time we're firing
                    _repeatCount++;

                    return Observable.timer(_repeatCount * POLLING_INTERVAL,
                            TimeUnit.MILLISECONDS);
                }
            });
        }
    })
    .subscribe(o -> _log("Executing polled task "),
            e -> Timber.d(e, "arrrr. Error"));
```

### 指数退避的重试机制

使用`retryWhen`操作符，实现延迟重试。其中延迟时间随retryCount的增加而增加
```
Observable.error(new RuntimeException("testing")) // always fails
    .retryWhen(new Func1<Observable<? extends Throwable>, Observable<?>>() {
        int _retryCount;

        @Override
        public Observable<?> call(Observable<? extends Throwable> observable) {
            return observable.flatMap(throwable -> {
                if (++_retryCount < MAX_RETRIES) {
                    return Observable.timer(_retryCount * RETRY_DELAY_MILLIS, TimeUnit.MILLISECONDS);
                }
                return Observable.error(throwable);
            })
    .subscribe(o -> _log("onNext"), e -> _log("fail, I give up"), () -> _log("onCompleted"));
```
延迟时间随retryCount的增加而增加，这样的能力还可以利用zipWith和flatMap来实现：
```
.retryWhen(observable -> observable.flatMap((Throwable throwable) -> {
            if (throwable instanceof UnknownHostException) {
                return Observable.error(throwable);
            }
            return Observable.just(throwable)
                    .zipWith(Observable.range(1, 5), (throwable1, i) -> i)
                    .flatMap(retryCount -> Observable.timer((long) Math.pow(5, retryCount), TimeUnit.SECONDS));
        }));
```

单纯的指数退避还可以通过delay实现：

```
Observable.range(1, 4)
        .delay(integer -> {
            // Rx-y way of doing the Fibonnaci :P
            return MathObservable
                  .sumInteger(Observable.range(1, integer))
                  .flatMap(targetSecondDelay -> Observable.just(integer)
                        .delay(targetSecondDelay, TimeUnit.SECONDS));
        })
        .subscribe(...);
```

### 表单验证（使用combineLatest）

```
_emailChangeObservable = RxTextView.textChanges(_email).skip(1);
_passwordChangeObservable = RxTextView.textChanges(_password).skip(1);
_numberChangeObservable = RxTextView.textChanges(_number).skip(1);
Observable.combineLatest(_emailChangeObservable,
      _passwordChangeObservable,
      _numberChangeObservable,
      (newEmail, newPassword, newNumber) -> {

          boolean emailValid = !isEmpty(newEmail) && ...;
          boolean passValid = !isEmpty(newPassword) && ...;
          boolean numValid = !isEmpty(newNumber) && ...;

          return emailValid && passValid && numValid;

      })
      .subscribe(formValid -> {
              if (formValid) {
                  _btnValidIndicator.setBackgroundColor(getResources().getColor(R.color.blue));
              } else {
                  _btnValidIndicator.setBackgroundColor(getResources().getColor(R.color.gray));
              }
      });
```

### 缓存机制：先从cache中获取数据再从网络拉取 (concat, concatEager, merge / publish)

```
// 使用concat实现先从磁盘拉取，再查询网络的缓存机制
Observable.concat(getSlowCachedDiskData(), getFreshNetworkData())
      .subscribeOn(Schedulers.io()) // we want to add a list item at time of subscription
      .observeOn(AndroidSchedulers.mainThread())
      .subscribe(...);

// 先从磁盘拉取，再从网络拉取，全部结束后将observable的值一次性按序发送出来
Observable.concatEager(getSlowCachedDiskData(), getFreshNetworkData())
      .subscribeOn(Schedulers.io()) // we want to add a list item at time of subscription
      .observeOn(AndroidSchedulers.mainThread())
      .subscribe(...);

// 还可以使用merge，但是merge不是按照原Observable的顺序，而是按照发射顺序，因此只适用于磁盘缓存一定快于网络的情况，否则会有磁盘数据未拿到最新更新的问题
...

// 使用 publish + merge 优化先加载缓存，再读取网络数据的请求过程。
// 先使用takeUntil操作符保证如果networkData的数据发送之后，diskData不再发送；
// 再用merge操作符将networkData和diskData数据发送合并；
// 由于takeUntil会订阅一次networdData，后面再订阅一次造成两次订阅，
// 为了两个订阅者都能收到发送的数据，使用publish操作符
getFreshNetworkData()//
      .publish(network ->//
                     Observable.merge(network,//
                                      getSlowCachedDiskData().takeUntil(network)))
      .subscribeOn(Schedulers.io()) // we want to add a list item at time of subscription
      .observeOn(AndroidSchedulers.mainThread())
      .subscribe(...);
```

### 使用 timer/interval/delay 实现任务调度

```
// 2s后执行一个任务并结束
Observable.timer(2, TimeUnit.SECONDS).subscribe(...);

// 每隔1s执行一次任务，初始延迟是1s
Observable.interval(1, TimeUnit.SECONDS).subscribe(...);

// 每隔1s执行一次任务，初始延迟是0
Observable.interval(0, 1, TimeUnit.SECONDS).subscribe(...);

// 每隔3s执行一次任务，5次后结束
Observable.interval(3, TimeUnit.SECONDS).take(5).subscribe(...);

// 先立刻执行一个任务，再间隔一定时间执行另一个任务
Observable.just("task A")
        .doOnNext(input -> _log(String.format("Doing %s right away", input)))
        .delay(1, TimeUnit.SECONDS)
        .doOnNext(oldInput -> _log(String.format("Doing Task B after a delay")))
        .subscribe(...);
```

### RxBus: (使用 RxRelay 和 debouncedBuffer)

RxBus的示例：

```
public class RxBus {
    private static RxBus mRxBus = null;
    
    // PublishSubject只会把在订阅发生的时间点之后来自原始Observable的数据发射给观察者，因此使用PublishSubject作为bus的内部实现。
    // SerializedSubject是线程安全的Subject
    private Subject<Object, Object> mRxBusObserverable = new SerializedSubject<>(PublishSubject.create());
    
    public static synchronized RxBus getInstance() {
        ...
    }
    public void post(Object o) {
        if (hasObservers()) {
            mRxBusObserverable.onNext(o);
		}
    }
    public Observable<Object> toObserverable() {
        return mRxBusObserverable;
    }
    public <T> Observable<T> toObserverable(Class<T> eventType) {
        // ofType操作符只发射指定类型的数据，其内部就是filter+cast
        return bus.ofType(eventType);
    }
    public boolean hasObservers() {
        return mRxBusObserverable.hasObservers();
    }
}

// 使用：发送事件
RxBus.getInstance().post(new StudentEvent("007","小明"));

// 使用：接收事件
rxSbscription = 
    RxBus.getInstance().toObserverable(StudentEvent.class)
        .subscribe(new Action1<StudentEvent>() {
            @Override
            public void call(StudentEvent studentEvent) {
                textView.setText("id:"+ studentEvent.getId()+"  name:"+ studentEvent.getName());
            }
        });
```

### 屏幕旋转时保存数据 (使用Subjects和retained fragment)

```
public class RotationPersist1WorkerFragment extends Fragment {

    private IAmYourMaster _masterFrag;
    private ConnectableObservable<Integer> _storedIntsObservable;

    @Override
    public void onAttach(Context context) {
        super.onAttach(context);
        _masterFrag = (IAmYourMaster) ...; // 找到宿主的引用，可以是activity可以是主fragment，用于监听Observable然后更新UI
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setRetainInstance(true); // Retain this fragment across configuration changes.
        
        // 构建Observable
        Observable<Integer> intsObservable = Observable.interval(1, TimeUnit.SECONDS)
                    .map(Long::intValue)
                    .take(20);

        _storedIntsObservable = intsObservable.replay(); // replay操作符，转换为一个connectable observable
        _storedIntsSubscription = _storedIntsObservable.connect(); // 触发事件发送
    }

    @Override
    public void onResume() {
        super.onResume();
        _masterFrag.observeResults(_storedIntsObservable); // 界面resume时通知给宿主UI，告知可以根据observable更新UI
    }

    @Override
    public void onDetach() {
        super.onDetach();
        _masterFrag = null; // 置空，防止内存泄露
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        _storedIntsSubscription.unsubscribe();
    }

    public interface IAmYourMaster {
        void observeResults(ConnectableObservable<Integer> intsObservable);
    }
}

public class RotationPersist1Fragment extends BaseFragment
      implements RotationPersist1WorkerFragment.IAmYourMaster {

    @Override
    public void observeResults(ConnectableObservable<Integer> intsObservable) {
        intsObservable.subscribe(...); // 宿主界面订阅Observable
    }
}

```



## Rx for Android

### RxAndroid

`AndroidSchedulers`是唯一留在RxAndroid的内容。即：

```
// 在主线程响应
Observable.just("one", "two", "three", "four", "five")
        .subscribeOn(Schedulers.newThread())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(/* an Observer */);
        
// 切换到任意Looper的线程
Looper backgroundLooper = // ...
Observable.just("one", "two", "three", "four", "five")
        .observeOn(AndroidSchedulers.from(backgroundLooper))
        .subscribe(/* an Observer */)
```

### RxLifecycle

```
// 使用RxLifecycle提供的组件
public class MyActivity extends RxActivity {
    @Override
    public void onResume() {
        super.onResume();
        myObservable
            .compose(bindToLifecycle())
            .subscribe();
    }
}

// 使用lifecycleProvider
public class MyActivity extends LifecycleActivity {
    private final LifecycleProvider<Lifecycle.Event> provider
        = AndroidLifecycle.createLifecycleProvider(this);

    @Override
    public void onResume() {
        super.onResume();
        myObservable
            .compose(provider.bindToLifecycle())
            .subscribe();
    }
}

// 使用bindUntilEvent
public class MainActivity extends RxAppCompatActivity {
    @Override
    protected void onResume() {
        super.onResume();
        Observable.interval(1, TimeUnit.SECONDS)
            .compose(this.<Long>bindUntilEvent(ActivityEvent.DESTROY))
            .subscribe(...);
    }
}
```

原理分析
RxLifecycle并不是真正解除订阅，而是在生命周期结束的部分取消了事件发送的序列。主要的实现方式是：先通过一个BehaviorSubject来发送生命周期事件，然后通过bindToLifecycle方法返回一个Transformer对象，使用原始的observable.compose(lifecycleTransformaer)，内部通过takeUntil操作符实现在特定 的生命周期事件发出后，原Observable的后续事件不再发送。

### RxBinding

```
RxView.clicks(view).subscribe(...);
RxView.longClicks(view).subscribe(...);
RxView.draws(view).subscribe(...);

RxView.visibility(btnClick).accept(true);
RxView.clickable(btnClick).accept(true);
RxView.enabled(btnClick).accept(true);

RxTextView.textChanges(etRxTextView).subscribe(...);

RxTextView.text(etRxTextView).accept("LeiHolmes");
RxTextView.hint(etRxTextView).accept("请输入姓名");
RxTextView.color(etRxTextView).accept(Color.parseColor("#00ff00"));

RxAdapterView.itemClicks(lvList).subscribe(...);
RxAdapterView.itemLongClicks(lvList).subscribe(...);
RxAdapterView.itemSelections(lvList).subscribe(...);

RxAdapterView.selection(lvList).accept(0));
```

### RxPermissions

```
final RxPermissions rxPermissions = new RxPermissions(this); // where this is an Activity or Fragment instance

// Must be done during an initialization phase like onCreate
rxPermissions
    .request(Manifest.permission.CAMERA)
    .subscribe(granted -> {
        if (granted) { // Always true pre-M
           // I can control the camera now
        } else {
           // Oups permission denied
        }
    });
    
// Must be done during an initialization phase like onCreate
RxView.clicks(findViewById(R.id.enableCamera))
    .compose(rxPermissions.ensure(Manifest.permission.CAMERA))
    .subscribe(granted -> {
        // R.id.enableCamera has been clicked
    });
    
rxPermissions
    .requestEach(Manifest.permission.CAMERA,
             Manifest.permission.READ_PHONE_STATE)
    .subscribe(permission -> { // will emit 2 Permission objects
        if (permission.granted) {
           // `permission.name` is granted !
        } else if (permission.shouldShowRequestPermissionRationale) {
           // Denied permission without ask never again
        } else {
           // Denied permission with ask never again
           // Need to go to the settings
        }
    });
    
rxPermissions
    .requestEachCombined(Manifest.permission.CAMERA,
             Manifest.permission.READ_PHONE_STATE)
    .subscribe(permission -> { // will emit 1 Permission object
        if (permission.granted) {
           // All permissions are granted !
        } else if (permission.shouldShowRequestPermissionRationale)
           // At least one denied permission without ask never again
        } else {
           // At least one denied permission with ask never again
           // Need to go to the settings
        }
    });
```

## References

- [x] [RxJava-Android-Samples](https://github.com/kaushikgopal/RxJava-Android-Samples)
- [x] [【译】对RxJava中.repeatWhen()和.retryWhen()操作符的思考](https://blog.csdn.net/feixiangsmile/article/details/80712267)
- [ ] [RxJava2 实战知识梳理(1~15) - 后台执行耗时操作，实时通知 UI 更新](https://www.jianshu.com/p/c935d0860186)
- [ ] [AndroidKnife/RxBus](https://github.com/AndroidKnife/RxBus) 
- [ ] [Blankj/RxBus](https://github.com/Blankj/RxBus)
- [ ] [ReactiveX/RxAndroid](https://github.com/ReactiveX/RxAndroid)
- [ ] [trello/RxLifecycle](https://github.com/trello/RxLifecycle)
- [ ] [JakeWharton/RxBinding](https://github.com/JakeWharton/RxBinding)