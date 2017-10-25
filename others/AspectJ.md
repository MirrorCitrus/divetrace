
[TOC]

# 概述
// TODO

# 在项目中使用AspectJ

在项目中集成AspectJ稍微有点麻烦。Jake大神的Hugo是使用Gradle插件的。

```
https://github.com/JakeWharton/hugo
```

详细的接入原理和接入方法参考References。

# AspectJ语法

## 概念
### Join Points

Join Points，简称JPoints，JPoints就是程序运行时的一些执行点。例如，构造方法调用、调用方法、方法执行、异常等等。AspectJ中的JPoints有如下一些：

- method call
- method execution
- constructor call
- constructor execution
- field get
- field set
- initialization
- static initialization
- handler
- ...

### Pointcuts

JPoint切入的位置。简单来说，Pointcuts的目标是提供一种方法使得开发者能够选择自己感兴趣的JoinPoints。


### Advice

Advice也就是我们具体插入的代码位置。包括：

- @Before
- @After
- @Around
- @AfterThrowing
- @AfterReturning

### Aspect

切面。Advice和Pointcut组合起来看做切面。

## Aspect语法

### Aspect语法的组成部分

```
@Before("execution(* android.app.Activity.on**(..))")
public void onActivityMethodBefore(JoinPoint joinPoint) throws Throwable {
}
```
这里会分成几个部分，我们依次来看：

- @Before：Advice，也就是具体的插入点。
- execution(...)：Pointcut语法。
- public void onActivityMethodBefore：实际切入的代码。

### Pointcut语法

每一种类型的JPoints有自己的筛选策略，称为Pointcut语法，表现形式类似于Java中的方法，有方法名和参数。方法名和JPoints类型一一对应，参数表示如何筛选感兴趣的JPoints。

#### Jpoint类型和Pointcut方法

JoinPoint类型 |	Pointcut语法
--|--
Method Execution(方法执行) |	execution(MethodSignature)
Method Call(方法调用) |	call(MethodSignature)
Constructor Execution(构造器执行)	| execution(ConstructorSignature)
Construtor Call(构造器调用)	| call(ConstructorSignature)
Class Initialization(类初始化)	| staticinitialization(TypeSignature)
Field Read(属性读) |	get(FieldSignature)
Field Set(属性写)| 	set(FieldSignature)
Exception Handler(异常处理) |	handler(TypeSignature)
Object Initialization(对象初始化) |	initialization(ConstructorSignature)
Object Pre-initialization(对象预初始化) |	preinitialization(ConstructorSignature)
Advice Execution(advice执行) |	adviceexecution()

#### Method/Constructor/Type/FieldSignature

常用通配符：

通配符	| 意义	| 示例
--|--|--
*	|   表示除”.”以外的任意字符串   |	java.*.Date：可以表示java.sql. Date,java.util. Date
..	|   表示任意子package或者任意参数参数列表   |	java..*:表示java任意子包；void getName(..):表示方法参数为任意类型任意个数
+	|   表示子类	|   java..*Model+:表示java任意包中以Model结尾的子类


- MethodSignature

```
[@注解] [访问权限] 返回值的类型 类全路径名（包名+类名）.函数名(参数)

示例：* *.*(..) // 通配所有方法
```

- ConstructorSignature

```
[@注解] [访问权限] 类全路径名（包名+类名）.new(参数)

示例：public *..People.new(..) // 表示任意包名下面People这个类的public构造器，参数列表任意
```

- FieldSignature

```
[@注解] [访问权限] 类型 类全路径名.成员变量名

示例：String com.example..People.lastName // 表示com.example包下面的People这个类中名为lastName的String类型的成员变量
```

- 间接对JPoint进行选择

除了上面表格当中提及到的直接对Join Point选择之外，还有一些Pointcut关键字是间接的对Join Point进行选择的。简单列举如下：

```
within(TypeSignature)	
withincode(ConstructorSignature/MethodSignature)
args(TypeSignature)	
```

- 组合Pointcut进行选择

Pointcut之间可以使用“&& | ！”这些逻辑符号进行拼接，将两个Pointcut进行拼接，完成一个最终的对JoinPoint的选择操作。

# References

- [x] [看AspectJ在Android中的强势插入](http://www.jianshu.com/p/5c9f1e8894ec)
- [ ] [深入理解Android之AOP](http://blog.csdn.net/innost/article/details/49387395)
- [x] [Android AOP学习之：AspectJ实践](https://juejin.im/post/58ad3944b123db00672cdeeb)