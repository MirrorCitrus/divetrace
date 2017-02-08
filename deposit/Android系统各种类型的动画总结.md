> 标签： "总结"

# 概述

为了让UI有着更好的交互效果，Andriod提供了各种各样设置动画的方法，这些方法使得Android的动画机制有着非常多的细分的点，因此大致地了解一下这些动画归属于哪里、什么时候会被触发、大概的原理是什么，不管对于写动画还是对于调试某些闪屏问题，都很有作用。

# 按实现方式分

按照实现方式来分的话，Android的动画大致可以分为这样三种：

* 帧动画：以Drawable为典型代表。这种动画的典型特点是，它的每一帧都定义了自己的持续时间，因此，有多少帧刷新多少次就可以了。
* 渐变动画：渐变动画的典型特点是：规定了一段时间内的from和to值，要求在这段时间内，自动按照某种频率刷新。渐变动画的实现框架又分为传统的View动画和属性动画。
* Surface动画：这个动画容易被忽视，指的是系统服务（其实就是WMS）直接对窗口的Surface进行的动画。我们有的时候会发现一个窗口明明不在同一个View当中，也能一起作动画；或者明明没有onDraw的调用，也明显产生了动画，其实就是因为系统在WMS侧添加了一个Surface动画。又因为Surface是不支持旋转的，所以我们在很多窗口过渡动画中，看不到旋转的效果。

# 按使用场景分

### 程序动画

没什么好说的，就是我们程序里面自己定义的动画。

### 窗口动画

窗口动画其实是WMS负责实施，又细分为三种:

* Activity的过渡动画，对应于AppWindowAnimator
* Window动画，对应于WindowStateAnimator
* 屏幕旋转动画，对应于ScreenRotationAnimator

我们先看一下AppWindowAnimator，这个主要是开放给我们定制Activity切换时候的动画的。我们可以通过重写Activity\#overridePendingTransition来定制：

```
 public class LeftRightSlideActivity extends Activity {
     @Override
     public void onCreate(Bundle savedInstanceState) {
         ...
         button.setOnClickListener(new View.OnClickListener() {
             @Override
             public void onClick(View v) {
                 Intent intent = new Intent();
                 intent.setClass(LeftRightSlideActivity.this, SlideSecondActivity.class);
                 startActivity(intent);
                 //设置切换动画，从右边进入，左边退出
                 overridePendingTransition(R.anim.in_from_right, R.anim.out_to_left);
             }
         });
     }
 }
```

再看一下WindowStateAnimator，我们显示Dialog，PopupWindow时经常打交道的一种动画。从应用侧的角度来说，可以通过setWindowAnimStyle来设置这个动画：

```
// 例如我们针对某个window调用这个方法
setWindowAnimations(R.style.mystyle);  //添加动画

// 然后在具体的style.xml中设置对应的动画。注意我们可以设置的item的name包含windowEnter/Exit
<resources>
    <style name="mystyle" parent="android:Animation">
        <item name="@android:windowEnterAnimation">@anim/dialog_enter</item>  //进入时的动画
        <item name="@android:windowExitAnimation">@anim/dialog_exit</item>    //退出时的动画
    </style>
</resources>
```

这样，app的过渡动画和窗口动画就设置成功了。我们发现，两种设置方式都是指定一个int类型的resource id来表征enter动画或者是exit动画，具体的动画参数则是和正常的View动画一样，支持alpha/scale/translate/rotate。另一方面，如果我们不设置，实际上也是会有默认的动画的，这意味着，如果我们需要禁用相关的动画，必须调用相关的方法，将resId设为0。

其实，这也和动画如何被使用的原理有关。简单来讲，我们设置的resId，会封装在前面指定的Animator中，这些Animator又会封装在WindowState中。我们知道，窗口的显示/移除/重新layout的时候，WindowState对象都会跨进程传递给WMS处理。WMS侧呢，会判断当前的窗口是哪种状态（Enter/Exit/Show/Hide\)，然后读取对应的resId，再从xml中载入Animation对象来进行动画。  
值得一提的是，动画载入并计算出正确的alpha和matrix后，要如何使用呢？这里并不是通过onDraw或者是操作canvas来实现动画绘制，而是直接调用SurfaceControl的setPosition/setAlpha/setMatrix等实现的。

### Transition动画

安卓5.0中Activity和Fragment变换是建立在名叫Transitions的安卓新特性之上的。这个诞生于4.4的transition框架为在不同的UI状态之间产生动画效果提供了非常方便的API。该框架主要基于两个概念：场景（scenes）和变换（transitions）。场景（scenes）定义了当前的UI状态，变换（transitions）则定义了在不同场景之间动画变化的过程。  
Transition框架实际上也是基于属性动画的，相当于是对Activity/Fragment以及其内部的View的一些动画进行了一些封装。Transition框架主要有这样几个优点：

* Transitions抽象和封装了属性动画，Animator的概念对开发者来说是透明的，因此它极大的精简了代码量。开发者所做的所有事情只是改变一下view前后的状态数据，Transition就会自动的根据状态的区别去生成动画效果。
* 不同场景之间变换的动画效果可以简单的通过使用不同的Transition类来改变，例如Fade，Slide等。

### 窗口Move动画

除了前面提到的几种动画之外，我们还经常发现窗口的大小变化之后会产生一个动画。这个动画自然还是在WMS添加的。具体的调用过程是这样的（注意入口是WMS.relayoutWindow）：
WMS.performLayoutLockedInner(...)
WMS.performLayoutAndPlaceSurfaceLockedInner(...)
WMS.performLayoutAndPlaceSurfaceLockedLoop(...)
WMS.performLayoutAndPlaceSurfaceLocked(...)

相关代码片段:
```
// WMS.java
private final void performLayoutAndPlaceSurfacesLockedInner(boolean recoveringMemory) {
     // ...
for (int displayNdx = 0; displayNdx < numDisplays; ++displayNdx) {
     // ...
final int N = windows.size();
for (i=N-1; i>=0; i--) {
WindowState w = windows.get(i);
// ...
                 final WindowStateAnimator winAnimator = w.mWinAnimator;
if (w.mHasSurface && w.shouldAnimateMove()) {
    // Frame has moved, containing content frame
    // has also moved, and we're not currently animating...
    // let's do something.
    Animation a = AnimationUtils.loadAnimation(mContext,
            com.android.internal.R.anim.window_move_from_decor);
    winAnimator.setAnimation(a);
    winAnimator.mAnimDw = w.mLastFrame.left - w.mFrame.left;
    winAnimator.mAnimDh = w.mLastFrame.top - w.mFrame.top;

    try {
        w.mClient.moved(w.mFrame.left, w.mFrame.top);
    } catch (RemoteException e) {
    }
}
}
}
}
```



