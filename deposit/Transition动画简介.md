# 概述
安卓5.0中Activity和Fragment变换是建立在名叫Transitions的安卓新特性之上的。这个诞生于4.4的transition框架为在不同的UI状态之间产生动画效果提供了非常方便的API。该框架主要基于两个概念：场景（scenes）和变换（transitions）。场景（scenes）定义了当前的UI状态，变换（transitions）则定义了在不同场景之间动画变化的过程。
Transition框架实际上也是基于属性动画的，相当于是对Activity/Fragment以及其内部的View的一些动画进行了一些封装。Transition框架主要有这样几个优点：

- Transitions抽象和封装了属性动画，Animator的概念对开发者来说是透明的，因此它极大的精简了代码量。开发者所做的所有事情只是改变一下view前后的状态数据，Transition就会自动的根据状态的区别去生成动画效果。
- 不同场景之间变换的动画效果可以简单的通过使用不同的Transition类来改变，例如Fade，Slide等。

# 使用
## 基本使用
* View属性变化的过渡
```
public class ExampleActivity extends Activity implements View.OnClickListener {
    private ViewGroup mRootView;

    private View mRedBox, mGreenBox, mBlueBox, mBlackBox;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ...
    }

    @Override
    public void onClick(View v) {
        TransitionManager.beginDelayedTransition(mRootView, new Fade());
        toggleVisibility(mRedBox, mGreenBox, mBlueBox, mBlackBox);

    }

    private static void toggleVisibility(View... views) {
        for (View view : views) {
            boolean isVisible = view.getVisibility() == View.VISIBLE;
            view.setVisibility(isVisible ? View.INVISIBLE : View.VISIBLE);
        }

    }
}
```
过程简单分析：

- 首先，点击按钮后调用了 beginDelayedTransition()， 将根场景和Fade Transition对象（淡入/淡出过渡效果）作为参数传递出去。框架立即对场景中所有 View 调用 Transitions 的 captureStartValues() 方法，同时， Transitions 将记录每个 View 的可见性。
- 调用结束后，开发者将场景中所有 View 设置为不可见的。
- 在下一个画面，框架对场景中所有 View(近期更新的) 调用 Transitions 的captureEndValues() 方法， Transitions 记录可见性。
- 框架调用 Transitions 的 createAnimator() 方法。Transition 分析每一个 View 的起始/结束状态，注意到 View 的可见性发生了变化。之后 Fade 对象利用这些信息创建了一个AnimatorSet 对象，并将其返回到框架中，进而将每个 View 的 alpha 值渐变到 0f。
- 框架运行返回的动画,让所有 View 从屏幕中淡出。

## Content Transition
在了解具体的ContentTransition之前，先了解两个API：

- Activity#overridePendingTransition(enterAnim:int, exitAnim:int)：这个方法允许我们给Activity的进入和退出，设置一个Transition动画
- FragmentTransaction#setCustomAnimation(enterAnim:int, exitAnim:int)：这个方法允许我们给Fragment的进入和退出设置动画

不过这两个API的问题在于，只能一次性给Activity/Fragment里面的所有View附着一个动画，而不能为其中部分View定制过渡动画，所以在5.0的时候Android系统提供了更加灵活的API。在认识这些API之前，先了解几个名词：

- 假设ActivityA/FragmentA启动了ActivityB/FragmentB,那么有如下四种动画可以定义：
    - ExitTransition:确定A启动B时A中View的动画
    - EnterTransition: 确定A启动B时B中View的动画
    - ReturnTransition: 确定B返回A时B中View的动画
    - ReenterTransition: 确定B返回A时A中View的动画
- 另外有两种类型的Transition可用：
    - ContentTransition
    - SharedElementTransition

我们首先认识一下ContentTransition。ContentTransition指的是activity/Fragment的过渡期间，两者的不共享View的过渡动画。

示例1：为Activity/Fragment整体设置Transition动画
我们可以稍微精简一下需要做的步骤：

- 首先我们需要设置window的属性为允许Transition动画：FEATURE_ACTIVITY_TRANSITIONS和FEATURE_CONTENT_TRANSITIONS（这一步也可以通过xml设置主题来实现）
- 然后，我们为当前Activity的Window设置Transition效果
- 启动新的Activity时，需要调用ActivityOptions.makeSceneTransitionAnimation(TransitionActivity.this).toBundle();
```
// TransitionActivityA.java
public class TransitionActivity extends Activity implements View.OnClickListener {

    private ViewGroup mRootView;

    @TargetApi(Build.VERSION_CODES.LOLLIPOP)
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

         getWindow().requestFeature(Window.FEATURE_ACTIVITY_TRANSITIONS); // 设置window的feature为允许Transition动画，必须在setContentView之前调用
         getWindow().requestFeature(Window.FEATURE_CONTENT_TRANSITIONS);

        getWindow().setExitTransition(new Fade());
        getWindow().setReturnTransition(new Slide());

        setContentView(R.layout.sub_activity_9);

        mRootView = (ViewGroup) findViewById(R.id.layout_root_view);
    }

    @TargetApi(Build.VERSION_CODES.LOLLIPOP)
    @Override
    public void onClick(View v) {
        switch (v.getId()) {
            case R.id.test_btn_2:
                Intent intent = new Intent(TransitionActivity.this, SubActivity.class);
                ActivityOptions option = ActivityOptions.makeSceneTransitionAnimation(TransitionActivity.this);
                startActivity(intent, option.toBundle());
                break;

        }
    }
}

// SubActivity.java
public class SubActivity extends Activity {
    @TargetApi(Build.VERSION_CODES.LOLLIPOP)
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        getWindow().requestFeature(Window.FEATURE_ACTIVITY_TRANSITIONS);
        getWindow().requestFeature(Window.FEATURE_CONTENT_TRANSITIONS);

        setContentView(R.layout.sub_activity_9_1);

        getWindow().setReenterTransition(new Slide());
        getWindow().setEnterTransition(new Explode());
    }
}
```
示例2：为具体的ViewGroup设置Transition动画
我们依然要精简一下步骤：

- 仍然是通过xml活着代码方式开启Feture_xxx_Transition
- 对于需要作为一个整体的ViewGroup，设置它的isTransitionGroup属性为true
- 为这个ViewGroup添加动画
```
/**
 * AndroidManifest.xml
 * 我们需要在manifest当中设置Activity的主题
 */
<activity
    android:name="com.alexjlockwood.activity.transitions.MainActivity"
    android:label="@string/app_name"
    android:theme="@style/AppTheme.Main">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
<activity
    android:name="com.alexjlockwood.activity.transitions.DetailsActivity"
    android:label="@string/app_name"
    android:theme="@style/AppTheme.Details" />

/**
 * themes.xml
 * 主题中，设置windowEnterTranstion/ReturnTransition/...,指向自定义的transition样式
 */
<resources>

    <style name="AppTheme.Main" parent="android:Theme.Material.Light.NoActionBar">
        <item name="android:statusBarColor">@android:color/black</item>
    </style>

    <style name="AppTheme.Details" parent="android:Theme.Material.Light.NoActionBar">
        <item name="android:statusBarColor">@android:color/black</item>
        <item name="android:windowEnterTransition">@transition/details_window_enter_transition</item>
        <item name="android:windowReturnTransition">@transition/details_window_return_transition</item>
        <!-- 我们还可以进一步设置android:windowSharedElementEnterTransition等动画 -->
    </style>

</resources>

/**
 * details_window_return_transition.xml
 * 我们通过设置transitionset的方式，设置具体的Transition动画
 */
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android"
    android:transitionOrdering="together"
    android:duration="500">
    <fade>
        <targets>
            <target android:excludeId="@android:id/statusBarBackground" />
            <target android:excludeId="@android:id/navigationBarBackground" />
        </targets>
    </fade>
    <slide android:slideEdge="top">
        <targets>
            <target android:targetId="@id/details_header_container" />
        </targets>
    </slide>
    <slide android:slideEdge="bottom">
        <targets>
            <target android:targetId="@id/details_text_container" />
        </targets>
    </slide>
</transitionSet>

/**
 * MainActivity.java
 * 我们在启动另一个activity的时候，需要指定相应的Bundle才可以让Transition动画生效。
 */
public class MainActivity extends Activity {
        @Override
        public void onClick(View v) {
            Intent intent = new Intent(MainActivity.this, DetailsActivity.class);
            intent.putExtra(EXTRA_STARTING_ALBUM_POSITION, mAlbumPosition);

            if (!mIsDetailsActivityStarted) {
                mIsDetailsActivityStarted = true;
                startActivity(intent, ActivityOptions.makeSceneTransitionAnimation(MainActivity.this).toBundle());
            }
        }
}
```

## SharedElementTransition
共享元素过渡（SharedElementTransition）决定了两个Activity/Fragment之间共享的View在场景切换时，具有怎样的过渡效果。使用共享元素动画有这样几个步骤：

- 开启window的feature(和前面一样)
- 设置具体的动画效果
- 指定共享元素。

我们先看第二步：设置动画效果。
有如下两种过渡效果可以定义（Window和Fragment都有此方法，所以我们使用的时候，在Activity.onCreate里面调用getWindow().set...即可）：

- setSharedElementEnterTransition(transition:Transition): 当我们在B中调用这个方法时，就定义了一个共享元素从A中到B中采用了怎样的变换
- setSharedElementReturnTransition(transition:Transition): 我们在B中调用这个方法，就定义了一个共享元素从B中返回到A中的变换

当然我们还是有exit/Reenter动画可以设置，但是基本上没有必要的。Android系统其实提供了几个预置的Transition，比较适合于共享元素Transition，它们是：

- ChangeBounds
- ChangeTransform
- ChangeClipBounds
- ChangeImageTransform
- 还有一个xml是融合了以上所有的。move（暂不知道如何使用）

但是实际上，我们完全没有必要设置任何一个，因为只要我们开启了FEATURE_ACTIVITY_TRANSITIONS，系统就已经为我们加载了预置的transition动画了。
我们再来看第三步：设置共享元素。
```
// 我们在使用makeSceneTransitionAnimation方法时，给定一个View对象和它对应的TransitionName(我们指定的),就可以指定共享元素了
Intent intent = new Intent(MainActivity.this, DetailsActivity.class);
startActivity(intent, ActivityOptions.makeSceneTransitionAnimation(MainActivity.this,
            mAlbumImage, mAlbumImage.getTransitionName()).toBundle());

// 当然我们还可以指定多个共享元素
ActivityOptions options = ActivityOptions.makeSceneTransitionAnimation(this,
        Pair.create(view1, "agreedName1"),
        Pair.create(view2, "agreedName2"));
```

## References
- [国外博客](http://www.androiddesignpatterns.com/2014/12/activity-fragment-transitions-in-android-lollipop-part1.html)
- [博文翻译](http://jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0113/2310.html)
- [官方教程](https://developer.android.com/training/transitions/index.html)















