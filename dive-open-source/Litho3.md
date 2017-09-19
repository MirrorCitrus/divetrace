[TOC]

# 概述

前一篇文章在调研Android UI性能优化时，发现对RecyclerView的性能优化是很多应用关注的一个点。很多应用都是一个基于RecyclerView的主界面，嵌套有不通业务逻辑的复杂视图。因此，有很多博文和开源框架在实现这样的主界面的组件化/结构，以及这种场景的性能优化。

Litho是Facebook今年开源的一个高效构建Android UI的声明式框架（declarative framework for building efficient UIs on Android）。Litho的出现可以追溯到Facebook去年的一篇博文 [Components for Android: A declarative framework for efficient UIs](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/1031/6721.html) 。Litho最初的目的是为了解决复杂列表的高效渲染和内存使用问题，Litho的四大特点也都是为了性能服务：异步布局、视图平铺、增量式挂载和细粒度的回收。而为了实现这四个特性，Litho放弃使用原生View，使用“组件”的方式展示界面元素，并使用自定义的View和布局来挂载这些组件。

不过，正是因为Litho完全放弃了原生View，因此目前来看，会受到这样一些因素的限制，比如界面调试时没有原生布局编辑器更加方便；视图在处理复杂逻辑时会不如原生控件方便；以及原生控件重新实现一套，使得库本身很重。因此Litho更适合对性能要求非常高的场景。

Litho已用于多个Facebook的安卓应用，例如Facebook、Facebook Lite、Messenger和Workplace等。 在界面滚动性能上，Facebook介绍Litho比安卓RecyclerView的性能提高了35%。

这篇文章只是对Litho做了初步的体验，和原理的简单探究。Litho在制作简单的UI时，利用注解实现编译期代码生成，因此定义组件基本都只需要关注业务，使用起来的感觉还是很不错的。不过实际的复杂开发中，体验好不好，也比较难评判。有兴趣的同学也可以自己尝试一下。

> 关于Litho如何使用，包括这一篇总共有三篇。前两篇基本是对照官方文档的翻译，做个参考。
[Litho(1) - 上手指南](http://agroup.baidu.com/share/md/a662d02bbb7c48ccb23b02d8e69a770c)
[Litho(2) - 进阶指南](http://agroup.baidu.com/share/md/677e4bb2e9a44cc1a30be7257326966c)
[Litho(3) - Litho分析（本篇）](http://agroup.baidu.com/share/md/3bbf4516d35945948a5361480c9c6191)


# 组件化

从直观来看，Litho展现了和前端非常类似的“组件化”技术在Android端的实现。当然，之所以要实现“组件化”，一方面是为了实现异步布局，另一方面是为了实现更加细粒度的组件复用，根本还是为了性能。虽然组件化不是Litho的初衷，我们也先看下Litho是如何实现组件化的。

## 组件的类型

Litho的组件分为两种，

- `LayoutSpec`: 将一些已知的组件组合成一个指定的布局，类似于ViewGroup
- `MountSpec`: 渲染一个View或Drawable的组件

### LayoutSpec
LayoutSpec相当于将原有的xml布局方式，改用了代码的方式来实现。传统的xml实际上帮助开发者有效地实现了业务逻辑和页面布局的分离，但是它的问题也很明显：一是性能不优，二是页面几乎没有逻辑绑定能力。后来Android引入了DataBinding框架，比较有效地弥补了二的问题，但是总还是不能像直接写代码那样自由。
而Litho使用组件的思路来连接逻辑和布局，组件可以接收外部传参的属性，也可以进行内部事件处理或状态处理，这使得组件具有比xml更强的逻辑处理能力，更加简洁和优雅一些。

```
@LayoutSpec
public class ListItemSpec {

  @OnCreateLayout
  static ComponentLayout onCreateLayout(ComponentContext c) {
  
    return Column.create(c)
        .paddingDip(ALL, 16)
        .backgroundColor(Color.WHITE)
        .child(
            Text.create(c)
                .text("Hello world")
                .textSizeSp(40))
        .child(
            Text.create(c)
                .text("Litho tutorial")
                .textSizeSp(20))
        .build();
  }
}
```

### MountSpec
MountSpec代表一个可以渲染View或者drawable的组件，实际上，Litho提供的各种基础组件，如`Text`，就是MountSpec实现的。使用Litho时，如果是想将已有的View/Drawable兼容进组件中，或者是像Litho一样实现一个原生控件，都需要利用到MountSpec。

```
@MountSpec
public class ColorComponentSpec {

  @OnCreateMountContent
  static ColorDrawable onCreateMountContent(ComponentContext c) {
    return new ColorDrawable();
  }

  @OnMount
  static void onMount(
      ComponentContext context,
      ColorDrawable colorDrawable,
      @Prop String colorName) {
    colorDrawable.setColor(Color.parseColor(colorName));
  }
}
```

## 组件依赖的数据：Props和States

组件的展现需要依赖于两个方面的数据：一种是外部提供的数据源，另一种是组件内部的状态。

Litho借鉴了React的思想：使用不可变输入和单向数据流机制。它使用的数据也包括两种类型：

- `prop`: 可以理解为组件需要外部提供的数据源，从父级继承并在整个View层级内传递。必须是immutable的。
- `state`: 可以理解为组件的内部状态，封装由Component管理并实现的细节，而对父级是透明的。必须是immutable的。

## 组件、组件树和组件生命周期

### 组件和组件树

LithoView持有一个组件树`ComponentTree`的成员变量，在Litho中，视图`LithoView`仅仅是一个宿主，它本身承载了一棵组件树的展示。组件树负责管理子组件（布局和展示），而并不依赖于Android原生的ViewGroup，这也是后面所说的“扁平化布局”的基础。

Litho中每个组件都是一个`Component<? extends ComponentLifecycle>`的实例。`ComponentTree`则管理一个根`Component`节点，它的主要工作是通过递归调用`onCreateLayout`的方式，来构建组件树；以及当有新的props时，触发对组件树的刷新。

`Component`大多数情况下，是组件的静态数据的封装，它的主要行为则主要由与之匹配的`ComponentLifecycle`来驱动。实际上，我们写的`LayoutSpec`和`MountSpec`都会生成对应的ComponentLifecycle，并由ComponentLifecycle来驱动。我们在`LayoutSpec`和`MountSpec`中使用Annotation标注的方法，实际上也是和ComponentLifecycle一一对应的生命周期方法。因此，组件的生命周期是这样的：

- `LayoutSpec`
	- `@OnCreateLayout`:  布局计算前执行一次，用于构建子组件的布局（类似于ViewGroup的功能）
- `MountSpec`
	- `@OnPrepare`: 布局计算前执行一次
	- `@OnMeasure`: 布局过程中调用
	- `@OnBoundsDefined`: 布局计算结束后调用一次
	- `@OnCreateMountContent`: 主要方法，创建内容，View/Drawable
	- `@OnMount`: 组件attach到宿主View前调用
	- `@OnBind`:组件attach到宿主View后调用
	- `@OnUnbind`：组件dettach到宿主View前调用
	- `@OnUnmount`：组件dettach到宿主View后调用

### Litho的渲染过程：布局和装载

在我们针对这里的生命周期再进行稍微细致的阐述之前，先了解一下Litho的渲染步骤：

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/8947f0c0be413d18abace4f006bd16e431064b12)

Android原生的View的渲染步骤包括三步：measure, layout, draw。
Litho的渲染步骤则是：Layout, Mount, Draw。

同样是三个步骤，但并不是一一对应的。其中，Litho的Layout过程，由LithoView的onMeasure方法触发，功能上对应原生View的measure和layout两个步骤，对组件树中的所有组件，包括LayoutSpec和MountSpec，进行测量和布局。Litho的Mount过程，则由LithoView的performLayout方法触发，对组件树种所有的MountSpec（注意装载过程已经不再有LayoutSpec的参与）进行Mount装载，主要处理View/Drawable的创建、回收和加工，类似于RecyclerView的bind过程。

### LayoutSpec/MountSpec的生命周期

了解了Litho的渲染过程之后，我们再看组件的生命周期：

- LayoutSpec

LayoutSpec所产生的Component，会经历三个生命周期方法：onCreateLayout, onPrepare, onMeasure，这三个方法都是从LithoView的onMeasure方法开始触发并执行完毕的。其中，LayoutSpec本身，只支持定义onCreateLayout，用于确定如何对子组件进行布局。在Litho的布局阶段，LayoutSpec会完成自身及所有子组件的测量和布局工作。

- MountSpec

MountSpec所产生的Component，会经历8个生命周期方法，都可以在MountSpec中定义具体的行为，且和ComponentLifecycle中的方法一一对应。这8个方法更具体的说明如下：

- Layout阶段
	- onPrepare(): LithoView的measure触发，每次布局前都执行一次
	- onMeasure(): 布局过程中调用。LithoView的onMeasure触发YogaNode的布局计算，过程中会触发onMeasure的调用
	- onBoundsDefined(): YogaNode布局计算完成后调用 
- Mount阶段
	- onCreateMountContent(): 为该类型的MountSpec创建装载的内容：View/Drawable，这里创建的内容不是MountSpec实例的成员变量，而是存放在公用的缓存池中，这也是组件精细回收的基础。
	- 	@OnMount, @OnBind, @OnUnbind, @OnUnmount: 都发生在Mount阶段，用于组件和具体展示的数据相关联或者解关联。

最后附上Litho的类图，作一个参考。整个结构和过程很简单：ComponentLifecycle负责控制生命周期，Component是根据MountSpec和LayoutSpec生成的组件，LayoutState负责控制布局阶段的操作，并将需要挂载的组件提取出来，封装成LayoutOutput给MountState进行挂载操作。挂载后生成MountItems封装需要绘制的drawable，由LithoView持有。

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/0f098d05f46ddd07f01cba107db15df6ae475b36)



# Feature/Architecture


## 异步布局

如前面所述，Litho中，组件的展示依赖的两种数据：props和state都是不可变的，这意味着组件的展示是线程安全的，这为使得异步布局成为可能。因此，Litho同时提供了同步的和异步的API用来做布局计算，两种API都是线程安全的并且都可以在任意线程中调用。

```
public class ComponentTree {
  // 同步API
  public void setRootAndSizeSpec(Component<?> root, int widthSpec, int heightSpec) {}
  // 异步API
  public void setRootAndSizeSpecAsync(Component<?> root, int widthSpec, int heightSpec) {}
}
```

开发者可以选择在一个ComponentTree上调用异步的API。另一方面，Litho提供的Recycler组件是默认开启异步布局的，异步API被广泛使用在RecyclerBinder中。这意味着，使用Litho构建滚动列表时，Litho可以提前计算好RecyclerView中item的布局，而不要阻塞UI线程。等到用户能看到item的时候，绝大多数繁重的工作都已经完成了。

我们回顾一下RecyclerView中展示组件的过程：

```
final RecyclerBinder recyclerBinder = new RecyclerBinder.Builder()
    .layoutInfo(new LinearLayoutInfo(this, OrientationHelper.VERTICAL, false))
    .build(context);
// add content
for (int i = 0; i < 32; i++) {
  recyclerBinder.insertItemAt(i, ComponentInfo.create()
									          .component(ListItem.create(context).build())
									          .build());
}
final Component component = Recycler.create(context)
    .binder(recyclerBinder)
    .build();
```
Recycler是litho提供的一个组件，对应于RecyclerView。Recycler的展示依赖于一个RecyclerBinder对象，这个对象兼具Adapter和LayoutManager的作用，同时负责数据填充和布局管理。

当我们在RecyclerBinder上发生数据的操作时（如insertItemAt），以及系统的onMeasure回调到来时，都会触发一次异步线程的布局计算。异步布局的主要工作是将未展现到屏幕的组件的布局信息提前计算好，在后续的measure时，尝试同步布局时可以直接将结果返回。

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/085b65003365743a39066eccaa5c6446ff446f8c)

## 视图平铺

我们知道，很多时候，过深的视图层级会带来性能问题，尤其是滚动列表，视图层级往往是性能的瓶颈。Litho很好地实现了视图的平铺，大大降低了布局层级。

我们可以看下面一个例子，这是一个命名为HeaderComponent的LayoutSpec，我们通过@OnCreateLayout方法，像写ViewGroup一样来定义它。但是打开开发者模式的“显示布局边界”，这里只有一个View，并没有嵌套层级。

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/db33dd73d6a3714bab4ca35a924cf1ae42b7f892)

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/52f6fbcb8fd5fdfb238e9b650031e1dafb90c1da)

Litho通过两点来实现这个效果：

-  第一，@OnCreateLayout布局返回的布局结构只是一个UI的轮廓，和安卓视图没有直接的关系。Litho的布局计算后，每个子组件的布局信息都可以被确定下来，Litho不会为LayoutSpec构建容器类，下一步的Mount过程也是完全跳过容器的概念的。

- 第二，Litho可以选择挂载一个view或者挂载一个drawable。事实上，对于Litho框架里的大多数核心组件，如Text和Image，挂载的是drawable，而不是view。

经过这些优化之后，HeaderComponent就可以作为一个单独的、完全扁平的视图被渲染。当然完全的扁平并不适用于所有的场景，有些依赖于Android原生View的功能会无法实现，比如子组件的事件处理、无障碍Accessibility能力，或者是局部的刷新。因此Litho有通用的“去扁平化”解决方案：例如，当为某个组件设置点击处理器时，该组件会自动被包裹在一个View中。

## 增量式挂载

虽然Component提供了扁平化的view层级结构和主线程外的布局计算功能，但是挂载（Mount）操作（包括对view和drawable的创建，回收和attach到宿主的工作）在复杂的component中依然花费主线程的大量资源，尤其是那些包含很多view的component。

Litho可以把装载component的花费平均分配至每一个UI帧来避免卡顿，并且对开发者来说是透明的。具体来说，使用Litho的增量式挂载（默认是开启的），LithoView将只会挂载与当前可见区域大小相适应的内容并且卸载和回收那些已经看不见了的component。与之相对应的是关闭增量式挂载，这意味着RecyclerView在布局时，会挂载全量的item内容。

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/badb62d136ef61a18f321a652352bb1902ef1c34)

## 细粒度的回收

RecyclerView提供了一种基于View type概念的回收机制，不同类型的item从不同的pool回收。绝大多数情况下这都能很好的工作，但是对于view类型非常多的列表就不理想了，因为RecyclerView需要不断的为每种类型inflating新的view，导致内存过度开销和滚动性能问题。

在Litho中，所有叶子控件（相当于MountSpec）如Text和Image在底层都是单独回收的。这可以让我们为RecyclerView带来更精细的回收方案。一旦一个组合控件的内部部分移出屏幕，该框架立即让它们可以被adapter中的其它item复用。

![图片](http://bos.nj.bpc.baidu.com/v1/agroup/78e88ccf0ba27a3a9b5771a8bacc86a0341b3f91)

这让我们更好理解MountSpec的生命周期：onCreateMountContent方法意味着为当前的MountSpec构建一个对应的可以挂载内容的对象，这个对象会直接放到全局的缓存池中以供复用。当这样的MountSpec需要被挂载时，从缓存池中获取，获取不到才会重新创建。mount方法发生在一个MountSpec组件进入屏幕的时候，当它需要被展现时，Litho框架会先从缓存池中取出一个Mount Content对象，再调用onMount挂载内容，再展现。反过来，onMount则发生在一个MountSpec组件移出屏幕的时候，当它不需要展现时，它会立刻调用onUnMount，并返回给缓存池。

每一个MountSpec的缓存池被封装成一个RecyclePool对象，每个MountSpec都被Litho按照递增的方式分配一个int类型的id，所有MountSpec的缓存池都由ComponentsPools管理：

```
/**
 * Pools of recycled resources.
 */
public class ComponentsPools {
  static final Map<Context, SparseArray<RecyclePool>> sMountContentPoolsByContext =
      new ConcurrentHashMap<>(4);
}
```
另外，ComponentsPools不仅管理MountSpec的缓存，还有很多Litho运行过程中所依赖的对象，都以对象池的形式被ComponentsPools管理，这也是Litho在内存优化上的一个很重要的一点。

# 最后

Litho使用了组件化的设计思路，给Android的UI构建方式又展示了一种新的可能。性能方面，Litho也做了几个大胆的尝试：异步布局、视图平铺、增量式挂载和细粒度的回收。组件化和性能，作为Litho的两个特点，还是让人感觉很惊喜的。

# References

- [x] [Litho:github](https://github.com/facebook/litho/blob/master/README.md)
- [x] [说一说Facebook开源的Litho](http://www.tuicool.com/articles/bQrMbaB)
- [x] [Facebook Litho：高性能安卓UI的构建框架](http://www.tuicool.com/articles/imeQFj7)
- [x] [文档翻译](https://shikieiki.github.io/categories/Android%E7%9F%A5%E8%AF%86/)
- [x] [Components for Android: 一个高效的声明式UI框架](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/1031/6721.html)