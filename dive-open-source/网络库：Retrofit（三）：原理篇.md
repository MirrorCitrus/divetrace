# 原理
抓住几个要点：
- 动态代理  
一个网络请求其实就是一个get方法，我们用注解描述requet，然后动态代理会帮助我们构建request,返回我们的其实是一个Call<T>对象，可以理解为一个可以get T in future的对象
解析response
- 线程转换  
如何转换到主线程：主要类：CallAdapter, MainThreadExecutor

## 源码分析
### 动态代理的思想和基本过程
Retrofit的create方法构建一个Service Interface的proxy：
```
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```
当我们调用这个方法的时候，实际上是这里的代理去执行，执行的步骤：
- 首先，根据当前方法的信息（注解、参数、返回值），构建一个ServiceMethod
- serviceMethod和实际的参数一起，构建一个实际的OkHttpCall
- 使用CallAdapter，对这个OkHttpCall进行类型转换和线程调度

### CallAdapter: Call<R>类型转换和线程调度
CallAdapter是一个将Call<R>转换为T的工具，同时，它也为线程调度提供了一个触发点。所以默认的（AndoridPlatform)产生的CallAdapter并没有处理类型转换，而是处理了线程调度。

```
public interface CallAdapter<T> {

  Type responseType();

  <R> T adapt(Call<R> call);
}
```
ServiceMethod在构建的时候，会向Retrofit去获取一个CallAdapter,而这个CallAdapter的实例是callAdapterFactories列表中首个满足条件的factory生成的。

```
public final class Retrofit {

  public CallAdapter<?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }
  
  public CallAdapter<?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
      ...
    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
    ...
  }
}
```
如果构建Retrofit的时候不添加任何CallAdapterFactory，那么默认会从当前的Platform里面获取。Platform是表征平台的基类，默认的CallAdapter什么都不做。对于Android来说，它会默认产生一个MainThreadExecutor作为adapt方法被执行的载体；再默认产生一个CallAdapter，这个方法并不做类型转换（即Call<R>被adapt后仍然是Call<R>），但是实际类型却是从OkHttpCall<R>转换成了ExecutorCallBbackCall<R>，后者会拦截enqueue方法的回调，通过MainThreadExecutor传递回主线程，再让外部的CallBack获得onResponse/onFailure的回调。

### ResponseConverter
获取方法和CallAdapter基本类似

### 参数解析细节

### 复习：Java动态代理

### RxJavaCallAdapter的原理



# 参考
- [ ] [Retrofit分析-漂亮的解耦套路](http://www.jianshu.com/p/45cb536be2f4)# 原理
抓住几个要点：
- 动态代理  
一个网络请求其实就是一个get方法，我们用注解描述requet，然后动态代理会帮助我们构建request,返回我们的其实是一个Call<T>对象，可以理解为一个可以get T in future的对象
解析response
- 线程转换  
如何转换到主线程：主要类：CallAdapter, MainThreadExecutor

## 源码分析
### 动态代理的思想和基本过程
Retrofit的create方法构建一个Service Interface的proxy：
```
public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object... args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            ServiceMethod serviceMethod = loadServiceMethod(method);
            OkHttpCall okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }
```
当我们调用这个方法的时候，实际上是这里的代理去执行，执行的步骤：
- 首先，根据当前方法的信息（注解、参数、返回值），构建一个ServiceMethod
- serviceMethod和实际的参数一起，构建一个实际的OkHttpCall
- 使用CallAdapter，对这个OkHttpCall进行类型转换和线程调度

### CallAdapter: Call<R>类型转换和线程调度
CallAdapter是一个将Call<R>转换为T的工具，同时，它也为线程调度提供了一个触发点。所以默认的（AndoridPlatform)产生的CallAdapter并没有处理类型转换，而是处理了线程调度。

```
public interface CallAdapter<T> {

  Type responseType();

  <R> T adapt(Call<R> call);
}
```
ServiceMethod在构建的时候，会向Retrofit去获取一个CallAdapter,而这个CallAdapter的实例是callAdapterFactories列表中首个满足条件的factory生成的。

```
public final class Retrofit {

  public CallAdapter<?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
  }
  
  public CallAdapter<?> nextCallAdapter(CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
      ...
    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
    ...
  }
}
```
如果构建Retrofit的时候不添加任何CallAdapterFactory，那么默认会从当前的Platform里面获取。Platform是表征平台的基类，默认的CallAdapter什么都不做。对于Android来说，它会默认产生一个MainThreadExecutor作为adapt方法被执行的载体；再默认产生一个CallAdapter，这个方法并不做类型转换（即Call<R>被adapt后仍然是Call<R>），但是实际类型却是从OkHttpCall<R>转换成了ExecutorCallBbackCall<R>，后者会拦截enqueue方法的回调，通过MainThreadExecutor传递回主线程，再让外部的CallBack获得onResponse/onFailure的回调。

### ResponseConverter
获取方法和CallAdapter基本类似

### 参数解析细节

### 复习：Java动态代理

### RxJavaCallAdapter的原理



# 参考
- [Retrofit分析-漂亮的解耦套路](http://www.jianshu.com/p/45cb536be2f4)