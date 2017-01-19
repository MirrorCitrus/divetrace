# 概述
> **CS:** Retrofit是对Http的Restful规则的封装，所以它既不关注底层的请求的实现，也不致力于提供更加便捷的操作。对于如何获得更方便、更灵活的请求结果的处理，则交给RxJava。Retrofit层面来说，它仅仅提供一个CallAdapter，来支持接入RxJava即可。
Retrofit封装请求后，会返回（也仅会返回）一个Call<T>，业务层面要如何处理T？是否要对T进一步转换为S？如何进行多轮请求？如何处理回调嵌套？这都是RxJava擅长的了。

在讲解实践经验之前，先看一下如何添加对RxJava的支持：只要为Retrofit添加一个CallAdapter即可：
```
Retrofit retrofit = new Retrofit.Builder()
        .baseUrl(baseUrl)
        .addConverterFactory(GsonConverterFactory.create())
        .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
        .build();
```
有了它，我们构建的ApiService返回的就可以是一个Observable了。

# RxJava和Retrofit的最佳实践经验
### 使用APIWrapper，封装请求细节
我们可以创建一个APIWrapper的类，将一个请求的操作进行封装（封装如何处理返回参数；是否对返回参数进行一轮或多轮的转换；是否进行连续的请求等），让外部关注最终的成功/失败的结果，并获取最终所需的返回对象。
```
public class APIWrapper {

    private Retrofit retrofit;
    private ApiService service;

    private APIWrapper() {
        retrofit = new Retrofit.Builder()
                ...
                .build();

        service = retrofit.create(ApiService.class);
    }

    private static class SingletonHolder {
        private static final APIWrapper INSTANCE = new APIWrapper();
    }

    public static APIWrapper getInstance(){
        return SingletonHolder.INSTANCE;
    }

    public void getTopMovie(Subscriber<MovieEntity> subscriber, int start, int count){
        service.getTopMovie(start, count)
                .subscribeOn(Schedulers.io())
                .unsubscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(subscriber);
    }
}
```

### 使用map，转换response的类型



### 自定义Subscriber，封装UI的反馈

# 参考
- [RxJava 与 Retrofit 结合的最佳实践](http://gank.io/post/56e80c2c677659311bed9841)
- [RxJava+Retrofit框架Demo（一）](http://www.jianshu.com/p/2b0aeb6b6b61#)
- [RxJava+Retrofit终极封装系列](http://blog.csdn.net/column/details/13297.html)