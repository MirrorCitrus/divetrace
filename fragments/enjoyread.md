# OverView

记录一些WalkThrough的简单内容和总结。

# 18) Epoxy

两种类型的组件：

- `EpoxyModel`：视图模型
- `EpoxyController`：控制视图如何添加到RecyclerView的Controller

#### ModelView的创建

有如下几种方案创建View的Model：
- 从自定义View创建Model: 使用`@ModelView`标注在自定义View上，并用`@Prop`注解标注需要接收的参数
- 从DataBinding创建Model：使用`@EpoxyDataBindingLayouts`来标注
- 从ViewHolders创建Model: 使用`@EpoxyModelClass`标注Class，使用`@EpoxyAttribute`标注属性，使用`@BindView`标注绑定

#### 在Controller中使用Model

Controller是用于控制不同的Model按照怎样的顺序、怎样的条件展示到RecyclerView上的。每个Model有个`addTo(..)`的方法，标志着把自己添加到对应的Controller中。而Controller有一个`buildModels()`方法，用于声明视图的展示。外部需要刷新RecyclerView时，调用`requestModelBuild`也会触发一次`buildModels()`的运行。

#### Model和Controller的实际使用

```
MyController controller = new MyController();
recyclerView.setAdapter(controller);
// Request a model build whenever your data changes
controller.requestModelBuild();
// Or if you are using a TypedEpoxyController
controller.setData(myData);
```

简易写法：

```
epoxyRecyclerView.setControllerAndBuildModels(new MyController());
// Request a model build on the recyclerview when data changes
epoxyRecyclerView.requestModelBuild();
```


#### References

- [ ] [Epoxy: Github](https://github.com/airbnb/epoxy)
- [ ] [Epoxy: Airbnb的安卓视图架构](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/1209/6838.html)

# 17) 模块化、组件化

组件化其实类似于一种架构模式，说明了要怎样组织工程结构。和插件化不同的是，每个插件都是可以独立运行的app，而组件是把大型工程拆分成一个个功能模块。所以组件最重要想解决的是：模块解耦、以及独立开发调试（提高效率）。更具体来说，主要包括以下一些内容：

- 代码解耦：
- 组件单独运行：每个组件增加测试Host模块，可以单独运行。具体包括两个步骤：1是将gradle中，把module的类型从library转换成application，2是通过编译开关，增加一些单独测试的代码、manifest和资源。
- 数据传递：cs架构，componentService+router，一般包括两个部分：页面路由和通用路由。其中四大组件的路由框架参考ARouter，主要是为了针对Android系统的xml定义的缺陷而定制的路由框架（重点是页面跳转）；而通用路由一般是要解决请求服务和结果返回的问题。
- UI跳转：其实就是前述提到的页面路由
- 组件生命周期：由于路由框架的componentService需要能寻找到组件，因此组件都需要在初始时注册服务，因此有生命周期的概念：onCreate注册自己；onDestroy解注册。
- 集成调试（按需编译）：组件可以编译成aar上传到本地仓库，主工程引用aar即可。
- 代码隔离：主工程的compile component的定义写到插件里，避免主工程直接引用组件的类或代码，实现完全的隔离。

一些参考资料主要详细解释了如何实现通用路由（包括多进程下的方案）

#### References

- [ ] [Android 模块化探索与实践](http://baronzhang.com/blog/Framework/Android-%E6%A8%A1%E5%9D%97%E5%8C%96%E6%8E%A2%E7%B4%A2%E4%B8%8E%E5%AE%9E%E8%B7%B5/)
- [ ] [Android组件化之通信（多模块，多进程）](http://www.jianshu.com/p/1fc5f8a2d703)
- [x] [Android彻底组件化方案实践](http://www.jianshu.com/p/1b1d77f58e84)

# 16) AspectJ

是一种AOP技术。核心概念是JPoint, Pointcut, Advice。集成到Android需要同时配置主工程和library的gradle，所以需要通过gradle插件的方式简化配置。

#### References

- [x] [看AspectJ在Android中的强势插入](http://blog.csdn.net/eclipsexys/article/details/54425414)
- [ ] [AOP 之 AspectJ 全面剖析 in Android](http://zhuhf.tech/2017/01/22/AspectJ/)
- [ ] [GitHub: gradle_plugin_android_aspectjx](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx/blob/master/README-zh.md)
- [x] [如何使用Android Studio开发Gradle插件](http://blog.csdn.net/sbsujjbcy/article/details/50782830)

# 13) Native崩溃的抓取

#### 1. 现有方案

- Google Breakpad: 跨平台；代码体量大
- 利用logcat: Android实现；不可靠
- coffee catch: 简洁、容易改动；兼容性问题

#### 2. 信号机制

- 程序崩溃和信号机制的基本概念
- 信号机制的过程
    - a) 内核收到信号后，将信号放入对应的进程的信号队列中，同时向进程发送一个中断，使其陷入内核态
    - b) 进程陷入内核态后，检测信号，发现有新信号后，进入信号处理过程。其中，检测信号的时机有两个:
        - 从内核态返回用户态之前
        - 从睡眠态被唤醒时
    - c) 处理信号：进程回到用户态，执行信号处理函数。执行完毕后还要回到内核态查看是否有信号待处理。全部处理完毕后，恢复到用户态和中断前的位置继续运行。
- 常见的信号量
    

#### 3. native crash捕获步骤

1) 注册信号处理函数

```
#include <signal.h> 
int sigaction(int signum,const struct sigaction *act,struct sigaction *oldact));
```

2) 设置额外栈空间
3) 兼容其他signal

#### 4. 堆栈的获取

- native堆栈获取
    - 4.1.1以上，5.0以下： libcorkscrew.so
    - 5.0以上：libunwind.so
    - 函数符号获取
- java堆栈获取
    - 前提：大多数情况下，crash线程就是捕获到信号的线程，虽然这在SIGABRT下不一定可靠。
    - 方法：在信号处理函数中获得当前线程的名字，然后把crash线程的名字传给java层，在java里dump出这个线程的堆栈，就是crash所对应的java层堆栈了。


#### References

- [x] [Android 平台 Native 代码的崩溃捕获机制及实现](https://mp.weixin.qq.com/s/g-WzYF3wWAljok1XjPoo7w) 

# 12) Glide
#### 快速使用：

```
Glide.with(this).load(url).into(imageView);
```
#### API详解

- 基础三步
    - `Glide.with():RequestManager` : 传入不同的context创建图片加载实例
        - Activity/Fragment/Application的context，内部根据具体context的生命周期处理加载的策略 
    - `load():DrawableTypeRequest<T>` : 加载图片
        - 加载本地图片: load(imgFile)
        - 加载应用程序资源: load(R.drawable.xxx)
        - 加载二进制流: load(imgByteArray)
        - 加载uri: load(imgUri)
    - `into():Target<T>` : 指定展示目标
        - into(imageView) 
- 占位图
    - `placeHolder(R.drawable.xxx):DrawableRequestBuilder<T>`
    - `error(R.drawable.xxx):DrawableRequestBuilder<T>`
- 缓存设置
    - `diskCacheStrategy(...)`: 设置缓存策略
        - `Strategy.ALL` : 
        - `Strategy.NONE` : 
        - `Strategy.SOURCE` : 
        - `Strategy.RESULT` : 
- 指定图片格式
    - `asBitmap()`
    - `asGif()`
- 指定尺寸
    - 默认自适应ImageView的大小
    - `override(100,100)`

#### 参考

- [ ] [github](https://github.com/bumptech/glide)
- [x] [Android图片加载框架最全解析（一），Glide的基本用法](http://blog.csdn.net/guolin_blog/article/details/53759439)
- [ ] [ Android图片加载框架最全解析（二），从源码的角度理解Glide的执行流程](http://blog.csdn.net/guolin_blog/article/details/53939176)
- [ ] [Google推荐的图片加载库Glide介绍](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0327/2650.html)


# 11) ConstraintsLayout
ConstraintsLayout是AndroidStudio2.2的新增功能之一，也是Google IO 大会上重点宣传的一个功能，特点是使用约束关系来指定控件的位置和大小，类似于更强大的RelativeLayout。结合AS的可视化操作，操作非常方便，不需要编写xml了。优点：布局规则更强，有效的减少布局嵌套。

要点：

- 约束系统：ConstraintsLayout的布局引擎 使用约束系统来约束控件的大小和位置
- Inspector: IDE提供给开发者进行布局的工具
- Handle的类型：
     - Resize Handle
     - Side Constraints Handle
     - Baseline Constraints Handle
- 创建约束的方式：
     - 手动创建约束
     - AutoConnect创建约束
     - Inference创建约束

#### 参考
- [ ] [代码实验室--带你一步步理解使用 ConstraintLayout](http://www.jianshu.com/p/793f76cf9fea)
- [ ] [Android新特性介绍，ConstraintLayout完全解析](http://blog.csdn.net/guolin_blog/article/details/53122387)

# 10) APK打包过程

#### 参考
- [ ] [自己动手生成Android Apk](http://blog.csdn.net/chenkai19920410/article/details/60589100)

# 9) Android AOP相关技术

## javaagent（java Instrumentation机制）
javaagent技术可以使我们在JVM加载class文件前对字节码作出修改。概括来讲，就是需要我们实现一个javaagent类（实现对字节码进行修改的方法），然后将该类以jar包形式输出，再在jvm中加载这个类。
由于javaagent技术对字节码修改时，利用的是Instrumentation.addTransformer方法，且java的instrumentation机制的本意也是通过agent来对java进程进行监控和协助。因此javaagent技术和java instrumentation技术应该是同一个意思

- 加载javaagent类
两种方法，一种是命令行方式添加，这种是jvm启动时就加载了，例如：`java -javaagent:/path/to/myagent.jar -jar myapp.jar
`
另一种是代码方式添加，这种属于jvm启动后的动态加载

- 实现javaagent
实现javaagent的premain或者agentmain方法，其中，方法的参数Instumentation很重要，它的addTransformer可以添加一个对字节码进行转换的对象。

基于上述方法，可以编写插件修改apk构建过程，进行字节码插入。大概的原理是这样的：应用构建过程中会启动一个dex任务，这个任务相当于在命令行执行了这样的命令：`java dx.jar com.android.dx.command.Main --dex --num-threads=4 —-output output.jar input.jar`,即启动一个java进程执行dx.jar的Main方法，这个方法本身就是把字节码打包成dex文件的，所以能找到字节码的hook点。我们通过命令行方式，给dx.jar添加javaagent，在hook点上获取字节码，再进一步修改这些类的字节码。
后来gradle plugin在1.5.0之后提供了Transform API，允许第三方插件修改Android应用的class文件了，大约流行的字节码插入的实现都是基于这个TransformAPI的。

## 字节码操纵技术

### ASM
ASM是一个java字节码操纵框架，它能被用来动态生成类或者增强既有类的功能。ASM与javassist相比，更贴近底层，较难使用但是效率更高。
ASM API按照visitor模式按照class文件结构依次访问class文件的每一部分。包括：ClassVisitor, MethodVisitor, FieldVisitor

- ClassVisitor
    - visit: 访问class类名，父类名，实现的接口等信息
    - visitField
    - visitMethod
- MethodVisitor
    - visitAnnotationDefault
    - visitCode: 访问方法体内的方法
    - visitLocalVariable
    - visitXxxInsn: 访问字节码指令（IADD, ARETURN...）

#### 参考
- [x] [JavaAgent使用](http://mp.weixin.qq.com/s/OljF6c8OpkctU5O4Vvf-2A)
- [x] [Android AOP之字节码插桩详解](http://www.yidianzixun.com/article/0FfQmNlM?share_count=2&from=groupmessage)

# 8) Dagger
#### Module和Component
Module: 生成Dependency的工厂
Comonent: Module的管理员
#### Component给Client提供Dependency的方法
方法一： 在Component里面定义一个返回Dependency的方法

1. 定义一个AppModule类，里面定义了一些Provider方法
```
 @Module
public class AppModule {
     // ...
    @Provides
    public LoginPresenter provideLoginPresenter(UserManager userManager, PasswordValidator validator) {
         return new LoginPresenter(userManager, validator);
     }
 }
```

2. 定义一个Component（接口形式），里面定义一个返回Dependency的方法。Dagger会根据从Module中生成这个dependency返回出来。
```
 @Component(modules = {AppModule.class})
 public interface AppComponent {
     LoginPresenter loginPresenter();
 }
```
3. 使用`DaggerAppComponent.builder().appModule(new AppModule(this)).build().loginPresenter();`获取一个Dependency对象


方法二：Field Injection

1. 在需要用到注入的client的位置，使用@Inject修饰需要注入的field
2. 为component添加和client类型匹配的inject()方法
```
@Component(modules = {AppModule.class})
public interface AppComponent {
    void inject(LoginActivity loginActivity);  //<=
}
```
3. 构建componenet，调用它的inject方法
```
public class LoginActivity extends AppCompatActivity {
    @Inject
    LoginPresenter mLoginPresenter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        AppComponent appComponent = DaggerAppComponent.builder().appModule(new AppModule(this)).build(); //<=
        appComponent.inject(this); //<=

        //从此之后，mLoginPresenter就被实例化了
        //mLoginPresenter.isLogin()
    }
}
```
4. Dagger自动从module中将所有被@Inject标注的field调用对应的provide方法注入

#### @Singleton和Constructor Injection
1. 如果希望Module里面的provide方法产生的实例是单例的，只要在方法上增加@Singleton注解
2. Module中如果很多provider方法里面都是直接new一个对象返回，那么这个provide方法就可以省略，在相应的类的构造函数上添加@Inject，这个和自己写一个provide方法是等价的

#### 使用Dagger进行单元测试
核心是将module进行spy,mock部分provide方法。

# 7) MVP/MVVM设计模式

#### 参考

- [x] [Android MVP详解（上）](http://www.jianshu.com/p/9a6845b26856)
- [ ] [Android MVP架构必要知识](https://zhuanlan.zhihu.com/p/25272412)


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
- [x] [Android单元测试（六）：使用dagger2来做依赖注入，以及在单元测试中的应用](http://chriszou.com/2016/05/10/android-unit-testing-di-dagger.html)
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
- [x] [Android自定义Lint实践](http://tech.meituan.com/android_custom_lint.html)
- [ ] [官方指南：使用 Lint 改进您的代码](https://developer.android.com/studio/write/lint.html?hl=zh-cn#manuallyRunInspections)
- [ ] [Building Custom Lint Checks in Android](https://www.bignerdranch.com/blog/building-custom-lint-checks-in-android/)

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


