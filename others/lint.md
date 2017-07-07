
## 概述

Android Lint是Google提供给Android开发者的静态代码检查工具。使用Lint对Android工程代码进行扫描和检查，可以发现代码潜在的问题，提醒程序员及早修正。

## Lint的运行和检查

// 略

## 自定义Lint检查

自定义包括如下几个步骤：

### 1. 创建自定义Lint的java工程，在gradle中配置lint 

```
dependencies {
    compile 'com.android.tools.lint:lint-api:24.3.1' // 官方给出的API
    compile 'com.android.tools.lint:lint-checks:24.3.1' // 已有的检查
}
```
### 2. 创建Issue

Issue由Detector发现并报告，是Android程序代码可能存在的bug。使用`Issue.create()`方法创建，包括如下几个参数：

- id: 唯一值，应该能简短描述当前问题。利用Java注解或者XML属性进行屏蔽时，使用的就是这个id。
- summary : 简短的总结，通常5-6个字符，描述问题而不是修复措施。
- explanation : 完整的问题解释和修复建议。
- category : 问题类别。Lint, Correctness, Security, Performance, Usability, Accessibility, Internationalization, Bi-directional text
- priority : 优先级。1-10的数字，10为最重要/最严重。
- severity : 严重级别：Fatal, Error, Warning, Informational, Ignore。
- Implementation : 为Issue和Detector提供映射关系，Detector就是当前Detector。声明扫描检测的范围Scope，Scope用来描述Detector需要分析时需要考虑的文件集，包括：Resource文件或目录、Java文件、Class文件。

### 3. 创建自定义的Detector 

- 继承Detector, 实现Scanner接口
    - Detector.XmlScanner
    - Detector.JavaScanner
    - Detector.ClassScanner
    - Detector.BinaryResourceScanner
    - Detector.ResourceFolderScanner
    - Detector.GradleScanner
    - Detector.OtherFileScanner 
- 复写`getApplicableNodeTypes()`方法，决定什么样的类型可以被检测到。常用的类型包括：
    - `MethodInvocation`
    - `ClassDeclaration`
- 复写`createJavaVisitor()`，创建Visitor，通常是`ForwardingAstVisitor`
- 复写`ForwardingAstVisitor`的`visitXxx()`方法，具体处理问题。常用的方法包括：
    - `visitMethodInvocation`
    - `visitClassDeclaration`
- 符合问题产生的条件时，使用`context.report(...)`上报，参数包括如下几种：
    - Issue： issue的实例
    - 当前节点
    - 当前位置信息，使用`context.getLocation(node)`获得
    - 添加的解释

### 4. 提供自定义的IssueRegistry，提供Issue列表

继承IssueRegistry，复写`getIssues():List<Issue>`方法，返回需要检测的issue的list。

### 5. 在gradle中声明Lint-Registry属性

```
jar {
    manifest {
        attributes("Lint-Registry": "com.meituan.android.lint.core.MTIssueRegistry")
    }
}
```

### 6. 打成jar包使用（详见References）

#### 方案1: 将jar拷贝到~/.android/lint中
#### 方案2: 将jar打到aar中


# References
- [x] [Android自定义Lint实践](http://tech.meituan.com/android_custom_lint.html)
- [ ] [官方指南：使用 Lint 改进您的代码](https://developer.android.com/studio/write/lint.html?hl=zh-cn#manuallyRunInspections)
- [ ] [Building Custom Lint Checks in Android](https://www.bignerdranch.com/blog/building-custom-lint-checks-in-android/)