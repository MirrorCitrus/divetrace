# 概述
> OkHttp: An HTTP & HTTP/2 client for Android and Java applications. OkHttp is efficient by default:
- HTTP/2 support allows all requests to the same host to share a socket.
- Connection pooling reduces request latency (if HTTP/2 isn’t available).
- Transparent GZIP shrinks download sizes.
- Response caching avoids the network completely for repeat requests.


# 拦截器和总体流程
OkHttp引入了拦截器机制，将一次网络请求的过程，转换为一个拦截器链的持续执行。让一个网络请求的过程既分工明确，又赋予开发者方便的定制和处理的能力。
拦截器Intercepter本身是一个接口，只有一个接口方法intercept()。利用这个接口，可以实现在request和请求后的response的观察和修改。
```
  Response intercept(Chain chain) throws IOException;
```
interceptor可以从chain中直接获取request，但是需要从chain.proceed方法中获取response，这意味着一个interceptor必须在它之后所有的拦截器都执行完之后，才能获取到response进行处理。由于一般拦截器链的最后都是进行真正的网络操作，所以用户的拦截器一般是在网络执行前对request进行操作、在网络执行后对response进行操作。

再回到Chain.proceed方法。拦截器链本身维护了一个拦截器列表，并保存一个index指向当前需要执行的拦截器，然后调用该拦截器的intercept方法。如果index指向了拦截器链的末尾，所有的执行也就结束。执行的触发点就是Chain.proceed，这也是刚刚所说，一旦一个拦截器调用了chain.proceed方法，它之后的拦截器会依次被调用执行，其后它才能拿到被处理过的返回。

OkHttp把用户的拦截器放在默认拦截器之前，默认拦截器链的构造在RealCall.getResponseWithIntereptorChain()中。也就是说，一个Call默认要经历这样5个拦截器，对应着一次网络请求的5个子过程：
![OkHttp拦截器过程图](/assets/okhttp-1.png)
- RetryAndFollowUpInterceptor: 
Retry和FollowUp，重试是指：当网络有问题、出错、超时等情况出现而导致错误时，通过再发起一次请求来进行重试；FollowUp是指：当需要授权、需要重定向、条件GET请求返回等情况发生时，需要接着进行一次请求。(StreamAllocation会在这里构造)。
- BridgeInterceptor：
桥接拦截器。主要是在请求前后进行一些Header的处理（主要是Cookie的一些处理）。
- CacheInterceptor：
负责缓存（磁盘缓存和Http缓存策略），核心在CacheStrategy当中。
- ConnectInterceptor：
连接操作。负责线路查找（Route查找，RouteSelector实现）、创建socket、握手、构建“输入/输出流”Source和Sink、并创建“解析器”Codec。另一方面，借助于ConnectionPool，实现连接复用，即：连接前，从ConnectionPool中查询是否有可复用的Connection；如果没有，新建的Connection要放入Pool中以被使用。
- CallServerInterceptor：
最后一层，进行Http流的读写和解析（也可以认为是序列化和反序列化）。借助于HttpCodec.writeRequestHeaders..等方法和Sink/Source实现。

# ConnectionPool机制：复用连接池和Connection自动回收
OkHttp的一个很大的特点就是连接复用以减少延迟（当然http2协议不需要），也就是说，默认为每个连接增加keep-alive头部。这也意味着，OkHttp需要管理可复用的连接，并在适当的时候关闭连接，实现的核心在ConnectionPool中。
另一方面，Keep-alive的头部的添加在前述的BridgeInterceptor中。而连接池的操作在前述的ConnectionInterceptor中，过程非常简单：
```
public final class ConnectInterceptor implements Interceptor {

  @Override 
  public Response intercept(Chain chain) throws IOException {
    ...
    HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```
首先调用streamAllocation.newStream(..)方法，这个方法负责查找或创建一个可用的连接Connection，再从Connection中获得对应的HttpCodec负责request/response的编码和解码，或者说序列化和反序列化。再从streamAllocation.connection()方法获取刚刚的可用连接，最后调用chain.proceed进行真正的request写入和response解析。
查找/创建一个可用连接的过程封装在StreamAllocation.findHealthyConnection(...)中，这个方法会先查找连接池中是否具有可复用的连接（ConnectionPool.get()），如果有可以直接返回；如果没有，则需新建，包括线路查找（Route查找，RouteSelector实现）、创建socket、握手、构建“输入/输出流”Source和Sink等过程。
讲完大致过程，再详细介绍一下ConnectionPool的工作机制。
- ConnectionPool的创建

ConnectionPool被OkHttpClient创建并持有。大概的结构如下：
```
public final class ConnectionPool {
  private static final Executor executor = new ThreadPoolExecutor(0 /* corePoolSize */,
      Integer.MAX_VALUE /* maximumPoolSize */, 60L /* keepAliveTime */, TimeUnit.SECONDS,
      new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp ConnectionPool", true));
      
  private final int maxIdleConnections;
  private final long keepAliveDurationNs;
  private final Runnable cleanupRunnable;

  private final Deque<RealConnection> connections = new ArrayDeque<>();
  
  public ConnectionPool() {
    this(5, 5, TimeUnit.MINUTES);
  }

  public ConnectionPool(int maxIdleConnections, long keepAliveDuration, TimeUnit timeUnit) {
    // ...
    }
  }
```
由默认的构造函数可知，默认的连接池最大有5个空闲连接，链路生命是5分钟。

- ConnectionPool的get/put API

既然是Pool，最重要的两个API自然就是get和put了。在StreamAllocation方法中，我们首先尝试从ConnectionPool中获取一个可复用的Connection，如果有，则直接返回；如果没有再创建，创建好之后，需要put到Pool中供复用。
值得一提的时，外部并不是直接通过ConnectionPool.get/put方法来操作的，而是通过Internal类的get/put方法，这两个方法其实就是对ConnectionPool.get/put方法做了一层包装，主要是为了让外部包的成员访问非public方法。

```
/**
* Returns a connection to host a new stream. This prefers the existing connection if it exists,
* then the pool, finally building a new connection.
*/
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
    boolean connectionRetryEnabled) throws IOException {
  synchronized (connectionPool) {
    // 从pool中获取Connection
    Internal.instance.get(connectionPool, address, this);
    if (connection != null) {
      return connection;
    }
  }
  // … 构建Connection
  synchronized (connectionPool) {
    // Pool the connection.
    Internal.instance.put(connectionPool, result);
  }
  return result;
}
```
由前述的ConnectionPool的结构可知，ConnectionPool通过 `connections:ArrayDeque<RealConnection>`来管理可复用的连接。
- Connection的自动回收

为了保证Connection在合适的时机（不再被使用、链路时间超过5分钟、同一个host下的连接超过5个）被自动关闭，需要有自动清理的逻辑。显然就是ConnectionPool的cleanUpRunnable所负责。大概的回收过程描述如下，代码就不贴了。
1. 每次put一个Connection时，都检查是否在运行cleanUpRunnable，如果没有运行就放入线程池中执行。这意味着只要ConnectionPool非空，这个runnable就在执行。
2. 遍历connections，找出其中没有被使用的、且空闲的时间最长的Connection，检查它是否满足(空闲socket连接超过5个&&keepalive时间大于5分钟)的条件，如果满足就清除、同时立刻重复2的操作；如果不满足条件，就等待这个连接的到期时间，也就是过一会儿再进行清除操作；如果都是活跃连接，则5分钟后再次进行清理操作。

- 基于引用计数法的连接活跃判定
  

# Cache & CacheStrategy：缓存策略

# Dispatcher：异步请求管理

# Sink&Source

# References
* [OkHttp3源码解析[综述]](http://www.jianshu.com/p/aad5aacd79bf#)
* [OkHttp3源码分析[复用连接池]](http://www.jianshu.com/p/92a61357164b)
* [OkHttp3源码分析[缓存策略]](http://www.jianshu.com/p/9cebbbd0eeab)
* [OkHttp3源码分析[DiskLruCache]](http://www.jianshu.com/p/23b8aa490a6b)
* [OkHttp3源码分析[任务队列]](http://www.jianshu.com/p/6637369d02e7)