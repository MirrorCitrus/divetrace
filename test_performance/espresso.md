# 概述

> The core API is small, predictable, and easy to learn and yet remains open for customization. Espresso tests state expectations, interactions, and assertions clearly without the distraction of boilerplate content, custom infrastructure, or messy implementation details getting in the way.

> Espresso tests run optimally fast! It lets you leave your waits, syncs, sleeps, and polls behind while it manipulates and asserts on the application UI when it is at rest.

# WalkThrough

## 快速上手

### 一看就懂的使用方法

```
@Test
public void greeterSaysHello() {
onView(withId(R.id.name_field)).perform(typeText("Steve"));
onView(withId(R.id.greet_button)).perform(click());
onView(withText("Hello Steve!")).check(matches(isDisplayed()));
}
```

### 构建

- 关闭模拟器或者设备的动画（包括窗口动画、过渡动画、程序动画）
- 确认下载了`Android Support Repository`
- 在app/build.gradle里面添加依赖：

```
androidTestCompile 'com.android.support.test.espresso:espresso-core:2.2.2'
androidTestCompile 'com.android.support.test:runner:0.5'
```

- 设置instrumentation runner: 在`android.defaultConfig`里面添加：

```
testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
```

- 可选的，关闭espresso默认的数据收集
- 示例： 

```
@RunWith(AndroidJUnit4.class)
@LargeTest
public class HelloWorldEspressoTest {

    @Rule
    public ActivityTestRule<MainActivity> mActivityRule = new ActivityTestRule(MainActivity.class);

    @Test
    public void listGoesOverTheFold() {
        onView(withText("Hello world!")).check(matches(isDisplayed()));
    }
}
```

### Basics

主要组件：

- `Espresso` : 我们通过Espresso提供的API和View交互（onView和onData)
- `ViewMatchers` : 
- `ViewActions` : 
- `ViewAssertions` : 

#### 使用`onView`查找View

onView方法接收ViewMatcher的参数，以便在View Hierarchy中确定唯一的View。而ViewMatcher则采用的是hamcrest匹配规则。

如果是简单地用R.id来查找View，可以使用：

```
onView(withId(R.id.my_view))
```
如果多个View共享同一个id的时候，这种写法可能会抛出`AmbiguousViewMatcherException`异常，并给出重复View的信息。

如果有多个相同id的View，我们可以使用多个条件做与的方式解决：

```
onView(allOf(withId(R.id.my_view), withText("Hello!")))
onView(allOf(withId(R.id.my_view), not(withText("Unwanted"))))
```

> Note: If the target view is inside an AdapterView (such as ListView, GridView, Spinner) the onView method might not work and it is recommended to use the onData method instead.

#### 在指定View上perform action

可以执行一次点击：

```
onView(...).perform(click());
```

可以实行多个操作：

```
onView(...).perform(typeText("Hello"), click());
```

如果一个View在一个`ScrollView`里面，可以先调用`scrollTo()`方法，将这个View展现到父View中：

```
onView(...).perform(scrollTo(), click());
```

#### 检查一个View是否满足断言

`check()`方法接收一个Assertion，检查一个View是否满足这个断言。最常用的assertion是`matches`。

```
onView(...).check(matches(withText("Hello!")));

```

#### 对AdapterView使用onData以控制ListView, GridView等

```
onData(allOf(is(instanceOf(String.class)), is("Americano"))).perform(click());
```

## Espresso Cheat Sheet

## Espresso Intents

## Espresso recipes

# References

- [ ] [Google Documentation](https://google.github.io/android-testing-support-library/docs/espresso/)