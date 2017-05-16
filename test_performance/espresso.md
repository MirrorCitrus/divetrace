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
 

# References

- [ ] [Google Documentation](https://google.github.io/android-testing-support-library/docs/espresso/)