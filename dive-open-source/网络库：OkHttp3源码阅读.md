# 概述

# 拦截器和总体流程
OkHttp引入了拦截器机制，将一次网络请求的过程，转换为一个拦截器链的持续执行。让一个网络请求的过程既分工明确，又赋予开发者方便的定制和处理的能力。
拦截器Intercepter本身是一个接口，只有一个接口方法intercept()。利用这个接口，可以实现在request和请求后的response的观察和修改。，
```
  Response intercept(Chain chain) throws IOException;
```
![OkHttp拦截器过程图](/assets/okhttp-1.png)
在外部没有附加任何额外操作的情况下，一个Call默认要经历这样5个拦截器，对应着一次网络请求的5个过程：

- RetryAndFollowUpInterceptor: 重试和FollowUp拦截器，作用：尝试从错误中恢复；尝试根据response进行授权、重定向、超时重试。(这个Interceptor会创建streamAllocation)
- BridgeInterceptor：桥接拦截器。首先对用户构建是request进行一定的转义与加工（主要是处理一下头部），然后获取response，最后对response做一些处理和加工后返回（主要是Cookie的一些处理）。
- CacheInterceptor：负责缓存，核心在CacheStrategy当中
- ConnectInterceptor：连接操作。负责线路查找（Route查找，RouteSelector实现）、创建socket、握手、构建“输入/输出流”Source和Sink、并创建“解析器”Codec。另一方面，借助于ConnectionPool，实现连接复用，即：连接前，从ConnectionPool中查询是否有可复用的Connection；如果没有，新建的Connection要放入Pool中以被使用。
- CallServerInterceptor：最后一层，进行Http流的读写和解析（也可以认为是序列化和反序列化）。借助于HttpCodec.writeRequestHeaders..等方法和Sink/Source实现。

# ConnectionPool机制：复用连接池和Connection自动回收

# Cache & CacheStrategy：缓存策略

# Dispatcher：异步请求管理

# Sink&Source

# References
* [OkHttp3源码解析[综述]](http://www.jianshu.com/p/aad5aacd79bf#)
* [OkHttp3源码分析[复用连接池]](http://www.jianshu.com/p/92a61357164b)
* [OkHttp3源码分析[缓存策略]](http://www.jianshu.com/p/9cebbbd0eeab)
* [OkHttp3源码分析[DiskLruCache]](http://www.jianshu.com/p/23b8aa490a6b)
* [OkHttp3源码分析[任务队列]](http://www.jianshu.com/p/6637369d02e7)