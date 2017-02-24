# OverView

记录一些WalkThrough的简单内容和总结

# 7) MVP/MVVM设计模式

#### 参考
- [ ] [目录](https://github.com/ChrisZou/android-unit-testing-tutorial)


# 6) 单元测试框架
单元测试框架包括：
- JUnit
- Mokito

Mokito的用法：
```
// 1. 验证方法是否被调用
UserManager mockUserManager = Mockito.mock(UserManager.class);  
Mockito.verify(mockUserManager).performLogin("xiaochuang", "xiaochuang password");  
// 2. 指定mock对象的某些方法的返回值
PasswordValidator mockValidator = Mockito.mock(PasswordValidator.class);
Mockito.when(mockValidator.verifyPassword("xiaochuang_is_handsome")).thenReturn(true);
// 3. 指定mock对象的某些方法的执行
Mockito.doAnswer(new Answer() {
    @Override
    public Object answer(InvocationOnMock invocation) throws Throwable {
        Object[] arguments = invocation.getArguments();
        NetworkCallback callback = (NetworkCallback) arguments[2];
        callback.onFailure(500, "Server error");
        return 500;
    }
}).when(mockUserManager).performLogin(anyString(), anyString(), any(NetworkCallback.class));
// 使用spy创建一个默认调用原来逻辑的mock对象
PasswordValidator spyValidator = Mockito.spy(PasswordValidator.class);
```
#### 参考
- [ ] [目录](https://github.com/ChrisZou/android-unit-testing-tutorial)
- [ ] [Android单元测试: 首先，从是什么开始](http://chriszou.com/2016/04/13/android-unit-testing-start-from-what.html)
- [x] [Android单元测试(三)：JUnit单元测试框架的使用](http://chriszou.com/2016/04/18/android-unit-testing-junit.html)
- [x] [Android单元测试（四）：Mock以及Mockito的使用](http://chriszou.com/2016/04/29/android-unit-testing-mockito.html)
- [x] [Android单元测试（五）：依赖注入，将mock方便的用起来](http://chriszou.com/2016/05/06/android-unit-testing-di.html) 
- [ ] [Android单元测试（六）：使用dagger2来做依赖注入，以及在单元测试中的应用](http://chriszou.com/2016/05/10/android-unit-testing-di-dagger.html)
- [ ] [Android单元测试（七）：Robolectric，在JVM上调用安卓的类](http://chriszou.com/2016/06/05/robolectric-android-on-jvm.html)
- [ ] [安卓单元测试(八)：Junit Rule的使用](http://www.jianshu.com/p/2cd745e54a78)
- [ ] [安卓单元测试(九)：使用Mockito Annotation快速创建Mock](http://www.jianshu.com/p/7f6a1d3aa516)
- [ ] [安卓单元测试(十)：DaggerMock, 让Dagger2与单元测试的结合易如反掌](http://www.jianshu.com/p/8c1b4ea379af)

# #5) 自定义Lint
Android Lint是Google提供给Android开发者的静态代码检查工具。为了满足一些工程个性化检查的需求，需要自定义Lint。
使用时，需要继承于Detector，然后实现Scanner接口。Scanner接口定义了检查范围，典型的是JavaScanner（还包括XmlScanner, ResourceFolderScanner等）。需要实现的方法包括：
- public List<Class<? extends Node>> getApplicableNodeTypes()： 用于指定需要检查的节点
- public AstVisitor createJavaVisitor(final JavaContext context)： 用于创建JavaVisitor
  - 例：ForwardingAstVisitor，复写visitMethodInvocation方法

创建Detector的同时，还需要创建对应的Issue，一个Issue将对应lint上报的信息展示。最后需要注册Issue。
##### 参考
- [Android自定义Lint实践](http://tech.meituan.com/android_custom_lint.html)

# #4) LeakCanary
LeakCanary.install()返回一个RefWatcher，用它监测某个对象时，会强制触发一次gc，再根据这个对象是否被回收判定是否有泄露。至于如何判定是否被回收，则是利用了WeakReference的一个特性：它指向的对象被回收时，引用会暂时存放在构建WeakReference的一个Queue中，查询该Queue即可判定是否被回收。

另一方面，LeakCanary.install(Application)方法默认监听所有Activity的泄露，原因是Application类中提供了registerActivityLifecycleCallbacks方法。
##### 参考
- [使用LeakCanary检测Android中的内存泄露](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0511/2861.html)
- [LeakCanary 内存泄露监测原理研究](http://www.jianshu.com/p/5ee6b471970e)

# #3) RecyclerView和ListView的缓存机制
ListView采用了二级缓存机制，RecyclerView则采用四级缓存机制，其中有一级缓存是开放给外部自定义的，因此大大提高了可扩展性。撇开这一层开放的缓存，RecyclerView的缓存优势主要在于从二级变到三级后，多出了一级负责减少bindView的调用。

另一方面，RecyclerView通过给每个Item增加flag的方式，在每次数据源变化时只更新需要更新的item，同样是减少bindView的调用。

##### 参考
- [Android ListView 与 RecyclerView 对比浅析--缓存机制](http://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653578065&idx=2&sn=25e64a8bb7b5934cf0ce2e49549a80d6&chksm=84b3b156b3c43840061c28869671da915a25cf3be54891f040a3532e1bb17f9d32e244b79e3f&scene=21#wechat_redirect)
- [RecyclerView必知必会](http://mp.weixin.qq.com/s/CzrKotyupXbYY6EY2HP_dA)

# #2) Android体系ID
* IMEI: Android手机的deviceId，需要权限
* 无线网卡地址: 
* ANDROID_ID: 每次恢复出厂设置生成的id
* 设备序列号（SN）: Build.SERIAL
* IDFA:
* OpenUDID:

# #1) Kotlin
##### 概览

##### 语言特性

* 和Java无缝对接
  * 集合框架全支持 + 扩展功能
  * 无缝互调
* 多样语法糖
  * 空安全
  * 属性代理
  * 扩展类
  * 函数式支持
  * 模式匹配
* 之于Android
  * 替换findViewById
  * 极简线程切换
  * （DSL方式）动态构建布局

##### 其他

anko库


