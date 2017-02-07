> MVVM（Model-View-ViewModel)，是最早由Microsoft提出的一种设计模式，在Windows上已经应用过较长一段时间。是继MVC, MVP之后的，又一种很受开发者推崇的设计模式。而实现MVVM的一个核心技术：数据绑定，也得到了Google官方的支持：Google I/O大会上，正式推出了Android上的DataBindingLibrary，这使得在Android应用上引入MVVM架构非常容易。

# MVVM模式简介
MVC（Model-View-Controller)和MVP(Model-View-Presenter)已经是非常流行的软件架构了。MVVM应该算是在MVP的基础上做了进一步扩展。下面一张图应该可以很清楚直观地反映这三种模式的区别：
![image](http://img.blog.csdn.net/20140401173412296?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbmFwb2x1bnlpc2hp/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  
可以看出，MVVM和MVP的区别，或者说，ViewModel的角色和Presenter角色的区别，在于和View的交互方式：MVP采用的是两个环节的交互（pass calls from View to Presenter & Presenter updates View)；而ViewModel直接采用一种叫做双向数据绑定的技术（Bi-direction data binding)。  
这篇文章其实更想介绍一下DataBindingLibrary的使用和实现原理。至于CleanArchitecture的探讨，应该是更深层次的东西，这里不作涉及。

# DataBinding Library Walkthrough
Android的DataBinding库是一个support库，在Android2.1(API level 7+)及以上都可以使用它。
## 构建环境
首先，需要在Android SDK Manager的支持库里下载该库。
然后，需要添加Data Binding到gradle构建文件里，如下：
```
android {
    ....
    dataBinding {
        enabled = true    
    }    
}
```

## 从一个例子开始
### 直接实现data binding的layout文件
我们需要在我们的xml文件中，通过简单的配置，指明View和我们的Model是如何关联起来的。很容易想象，我们会需要指定，它关联的Model的类名是什么，哪些视图关联到其中的哪些成员变量中。  


```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.name}"/>
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.say}"/>
   </LinearLayout>
</layout>
```


我们分析一下上述的xml文件：
- 首先xml文件的顶层标签不再是父视图，而是一个<layout>标签。原有的视图内容作为子标签，和原来一样编写即可。
- 有一个<data>的子标签，内部用于标识当前视图需要绑定的model的信息
    - <variable>标签表示了我们要绑定的一个数据对象。它的类型（type属性）是com.example.User，它在当前xml文件中的变量名称（name属性）是user
- 在具体的布局中，我们除了为属性写直接值，写@string/这种资源引用值以外，还可以写属性表达式：`@{}`。这个表达式将带领我们直接获取到model的内容，比如我们写`@{user.firstName}`，可以得到一个User类对象的firstName成员变量的值。

### Java层面的POJO类作为Model
另一方面，这里的User，是一个POJO，这个很好理解。

### 绑定数据
我们需要把我们定义的xml文件inflate出来，并且给这个layout指定一个具体的类对象。我们需要这样：
```
@Override
protected void onCreate(Bundle savedInstanceState) {
   super.onCreate(savedInstanceState);
   MainActivityBinding binding = DataBindingUtil.setContentView(this, R.layout.main_activity);
   User user = new User("Citrus", "Hello World");
   binding.setUser(user);
}
```
- 首先，任意一个这样的layout文件，DataBinding库会帮助我们配合生成一个Binding类。类名是通过[Pascalcase规范后的layout名]+Binding形成。例如：main_activity.xml会生成MainActivityBinding。
- 另一方面，DataBinding库提供了DataBidingUtil这个工具类，帮助我们加载xml并生成XxxBinding的实例。
- 最后，我们要通过biding.setXxx来设置具体的model对象。

当然，我们除了DataBidingUtil这个类，还可以通过其他方式加载xml和获得binding对象，如：

```
// 直接通过具体Binding类的inflate方法
MainActivityBinding binding = MainActivityBinding.inflate(getLayoutInflater());

// ListView或者RecyclerView的adapter使用Data Binding
ListItemBinding binding = ListItemBinding.inflate(layoutInflater, viewGroup,
false);
//or
ListItemBinding binding = DataBindingUtil.inflate(layoutInflater, R.layout.list_item, viewGroup, false);

```

### 事件处理和MethodBinding
除了绑定数据，还可以绑定事件（进一步，还可以绑定提供数据的方法）。例如：

```
 <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{MyStringUtils.capitalize(user.lastName)}"
           android:onClick="@{handlers::onClickFriend}"/>
```

 
> 更详细的内容参考官方指南，这里不作赘述。

# DataBindingLibrary的实现原理

数据绑定的核心任务大约可以这样表述：
- **初始化绑定**：一个Model对象和一个视图（或者是视图树）绑定时，能够把Model的属性正确地展示到ViewTree上；
- **变化的双向通知和Rebind**：View和Model的任何一方发生变化，都可以自动影响到另一方，使另一方对应的属性发生变化。很遗憾的是，DataBindingLibrary目前还只支持单向绑定，即：Model的变化自动影响到View的展示（不过其实应该也足够用了）。  
所以，这一节我们从初始化绑定、Model的变化通知到View两个个方面来分析一下DataBindingLibrary的原理。其他的细节不展开分析。

## 初始化绑定
初始化绑定要解决这样两个问题：
- 如何找到View？
- 如何建立和实现View的属性和Model属性的映射？  

第一个问题，如何找到View，我们从例子中可以看出，我们制作一个简单界面时，没有用到任何的findViewById，用过ButterKnife的同学明白，单是省去了findViewById这一句话，就已经非常方便了。从中我们也可以理解，dataBinding在ButterKnife的基础上，还帮助我们省去了set/get/change等操作，开发起来会非常快捷方便。  
第二个问题，其实也牵涉两个问题：一个是，一个Model的哪个属性，映射到哪个View的哪个属性上，这个映射我们如何建立；另一个是，找到了映射，我们用什么方法（method)来设置？简单想，肯定是setText，setPadding，setOnClickListener这些方法都应该支持。

#### 总体过程
ok，我们从代码里面找。起点当然是DataBindingUtils.setContentView，这里面的过程一目了然，大概用图来表述是这样的：
![初始化绑定过程.png](http://upload-images.jianshu.io/upload_images/1767465-f5cfab19f6ebfa79.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在讲解图中过程之前，先要了解，DataBinderMapper类和ActivityMainBinding类都是编译期生成的类，源代码在工程的./app/build/generated/目录下。正是这些代码自动帮助我们实现了和业务逻辑紧密相关的ViewModel，但是却没有需要我们动手写任何一行逻辑。  
![编译期生成的Java代码](http://upload-images.jianshu.io/upload_images/1767465-2c2ec62f5923e4e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
回到DataBindingUtils.setContentView方法过程：先调用activity的setContentView，然后调用DataBinderMapper类的getDataBinder方法。DataBinderMapper是编译期生成的类，作用是做一个layout文件id和具体的XxxBinding类的映射，从而调用对应的XxxBinding类的bind方法。代码是这样的：
```
class DataBinderMapper {
    ...
    public android.databinding.ViewDataBinding getDataBinder(android.databinding.DataBindingComponent bindingComponent, android.view.View view, int layoutId) {
        switch(layoutId) {
                case com.citrus.demo.R.layout.activity_main:
                    return com.citrus.demo.databinding.ActivityMainBinding.bind(view, bindingComponent);
        }
        return null;
    }
```
不管是调用DataBindingUtils.setContentView还是AcitivityMainBinding.inflate方法，都调用的是AcitivityMainBinding.bind方法，这个方法做了两件事：首先new一个ActivityMainBinding的实例；然后通过invalidateAll()方法的调用，最终调用到AcitivityMainBinding.excuteBinding方法。其中，在构建Binding实例的过程中，完成了对View的查找；在调用executeBinding方法时，真正实现了数据的绑定。下面详细分析下这两个过程。

#### ActivityMainBinding的构造方法
在讲解ActivityMainBinding的构造方法之前，注意到DataBinding库其实对我们的xml文件做了一层预处理。具体来讲，预处理包括两步：1）将View树的层级还原成普通的layout文件，去除里面的属性表达式，为涉及DataBinding的View节点设置tag。2）将<data>标签抽离，生成一个`[layout文件名]-layout.xml`的文件。  
原来的xml文件是这样的：

```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <data>
        <variable name="user" type="com.citrus.demo.User" />
    </data>

    <LinearLayout
        .../>

        <TextView
            android:id="@+id/user_name"
            ...
            android:text="@{user.name}" />

        <TextView
            android:id="@+id/user_say"
            ...
            android:text="@{user.say}" />

        <Button
            ...
            android:text="@{user.name}"/>
    </LinearLayout>

</layout>
```
预处理后，xml文件变成这样：

```
<LinearLayout
        android:tag="layout/activity_main_0"
        .../>

        <TextView
            android:id="@+id/user_name"
            android:tag="binding_1"
            ... />

        <TextView
            android:id="@+id/user_say"
            android:tag="binding_2"
            ... />

        <Button
            android:tag="binding_3"
            ...
            android:text="@{user.name}"/>
    </LinearLayout>
```
我们看到，根视图和所有绑定了数据的View节点（其实应该还包括所有有id定义的节点）,都被设置了tag。tag的命名规则是这样的：
- 根视图以“layout/”开头，接上当前xml的文件名，再加上下划线和一个编号0。
- 子视图统一以“binding_”开头，后面跟一个序号。序号接着之前的0开心，依次往下排。


然后我们再看初始化的代码：

```
public class ActivityMainBinding extends android.databinding.ViewDataBinding  {

    // views
    public final android.widget.LinearLayout activityMain;
    private final android.widget.Button mboundView3;
    public final android.widget.TextView userName;
    public final android.widget.TextView userSay;
    // variables
    private com.citrus.demo.User mUser;

    public ActivityMainBinding(android.databinding.DataBindingComponent bindingComponent, View root) {
        super(bindingComponent, root, 1);
        final Object[] bindings = mapBindings(bindingComponent, root, 4, sIncludes, sViewsWithIds);
        this.activityMain = (android.widget.LinearLayout) bindings[0];
        this.activityMain.setTag(null);
        this.button = (android.widget.Button) bindings[3];
        this.userName = (android.widget.TextView) bindings[1];
        this.userName.setTag(null);
        this.userSay = (android.widget.TextView) bindings[2];
        this.userSay.setTag(null);
        setRootTag(root);
        // listeners
        invalidateAll();
    }
    ...
}
```
首先看到，在注释`// views`下面，都是View类型的成员变量，而这些成员变量恰恰对应到activity_main.xml文件里的视图节点。注意到命名规则大概是这样的：
- 根节点LinearLayout未设置id，因此代表根节点的成员变量直接使用xml的名称命名：activity_main => activityMain:LinearLayout
- 每个涉及绑定的子节点都会生成一个对应的成员变量，如果该子节点设置了id，则直接使用该id为成员变量命名。例如我们为表征name的TextView设置id：`android:id=“@+id/user_name”`，此时会使用驼峰法生成userName这个成员变量。
- 如果子节点没有设置id，那么会生成一个类似于`private final android.widget.Button mboundView3`这样的成员变量。可以看到，它已经不是public的了。`mboundView`是固定的前缀，后面的3是怎么来的呢？回顾刚才预处理过的xml文件即可得知：这个3正是赋予的tag最后的编号。

然后，在注释`// variables`下面，是当前layout文件里，<data>标签的子标签<variables>里面定义的变量。其实表示的是Model。命名规则应该是在variables标签的name前面添加`m`，例如`<variables name="user" .../>`被命名成mUser。

接下来我们看构造函数。首先，调用了ViewDataBinding的静态方法mapBindings，传入的参数包括：1）DataBindingComponent；2)当前的根视图root；4）常量4，这个4其实就是预处理xml的时候，数出的需要绑定的View节点的数量；5）sIncludes；6）sViewsWithIds。
这个方法里面做的事情其实也很简单：先构造一个长度为4的Object数组，然后从root开始，递归调用另一个mapBindings的重载方法，依次判断所有的view节点应该存放在数组的哪个位置，存放好之后返回。直观地来讲：这个方法的作用就是把预处理时，按照设置的tag最后标识的序号，将View对象排列好形成数组返回出来。

返回了bindings数组之后，根据序号的对应关系，将View实例赋值给成员变量，最后清空预处理时赋予的tag即可。

分析到此可知，通过预处理=>代码生成=>XxxBinding的构建，我们实现了findView这个任务（也就是最开始提出的第一个问题）。

#### ActivityMainBinding.executeBinding方法
当然，在最初的流程中，executeBinding是在构建Binding实例时就被调用到的，但是其实这个时候我们并没有设置绑定的Model，所以调用了excuteBinding方法也是没有任何效果的。只有我们真正调用setUser方法之后才能得到绑定的数据。这个过程我们暂时略过，总之最后还是进入executeBindings方法。  
看下精简后的方法的代码:

```
public class ActivityMainBinding extends android.databinding.ViewDataBinding  {

    // views
    public final android.widget.LinearLayout activityMain;
    public final android.widget.Button button;
    public final android.widget.TextView userName;
    public final android.widget.TextView userSay;
    // variables
    private com.citrus.demo.User mUser;
    
    ...
    
    @Override
    protected void executeBindings() {
        
        java.lang.String nameUser = null;
        java.lang.String sayUser = null;
        com.citrus.demo.User user = mUser;
        
        if (user != null) {
            nameUser = user.name; // read user.name
        }
    
        if (user != null) {
            sayUser = user.say; // read user.say
        }
    
        android.databinding.adapters.TextViewBindingAdapter.setText(this.userName, nameUser);
        android.databinding.adapters.TextViewBindingAdapter.setText(this.userSay, sayUser);
    }
}
```
整个过程也是非常清晰：首先，为每个表达式的返回值构建一个临时变量。临时变量的命名是从表达式本身衍生的，例如`@{user.name}`会生成一个`String nameUser`的变量;  
其次，从Model中获得属性值。例如：表达式`@{user.name}`表示，我可以直接利用`user.name`获取属性值；如果我们给name这个成员变量设置了getter，那么DataBindingLibrary则会使用`user.getName()`来取值；  
最后，使用`TextViewBindingAdapter.setText(this.userName, nameUser);`这个方法，来将这里的属性值，设置到对应的TextView上去。  
我们可以看出，这段代码实际就完成了Model的属性和View的属性的映射。即：user.name <=> userName:TextView的setText方法。

我们再看一下TextViewBindingAdapter.setText方法：
```
public class TextViewBindingAdapter {

    ...

    @BindingAdapter("android:text")
    public static void setText(TextView view, CharSequence text) {
        final CharSequence oldText = view.getText();
        if (text == oldText || (text == null && oldText.length() == 0)) {
            return;
        }
        if (text instanceof Spanned) {
            if (text.equals(oldText)) {
                return; // No change in the spans, so don't set anything.
            }
        } else if (!haveContentsChanged(text, oldText)) {
            return; // No content changes, so don't set anything.
        }
        view.setText(text);
    }
}
```
这里就需要介绍一下BindingAdapter机制。正常来讲，当我们设置了绑定`android:text=@{...}`，我们可以想到一种简单的设置属性值给View的方法，反射调用setXxx，例如这里，我们只要反射调用setText即可。但是这种做法往往不尽如人意，或者说，缺乏更加灵活的介入机制。因此，DataBinding提供了BindingAdapter机制，让我们可以指定哪一种`android:xxx`属性可以映射到哪一个方法当中。同时，DataBindingLibrary也提供了预置的类来实现这套机制。

# To Be Continued
这篇文章主要是从业务层面分析了DataBinding的原理和运作过程。所谓“业务层面”是指：完全和业务相关部分的代码。所以分析的大多是编译期生成的AvtivityMainBinding、BR的原理。其实还有很多地方值得继续看一下。例如：原理层面：DataBinding是怎样利用apt直接帮我们生成业务代码的？`@BindingAdapter`注解是如何被处理的？xml文件的预处理和属性表达式都是如何处理的？等等。还有应用层面：进阶用法？RoboBinding是如何实现双向绑定的？如何自定义BindingAdapter等等。不过这次的学习就暂时告一段落吧。

# 参考文档
- [Data Binding Library Guide](https://developer.android.com/topic/libraries/data-binding/index.html)
- [Data Binding（数据绑定）用户指南](http://www.jianshu.com/p/b1df61a4df77)
- [对MVC、MVP、MVVM的理解](http://blog.csdn.net/napolunyishi/article/details/22722345)
- [DataBinding实现原理探析](http://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650237550&idx=1&sn=e0a6955a8ceb80895fd89ade3b8d8800&chksm=88639b01bf1412178f3dcd79fecfed6d8deedd0195b70edbd0251d3889df5bbe7242bba571b3&mpshare=1&scene=1&srcid=1102TzSg19mX2kSlrRkEiXI5#rd)

