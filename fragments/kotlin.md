# OverView

记录一些WalkThrough的简单内容和总结

# RecyclerView和ListView的缓存机制 (20170120)
ListView采用了二级缓存机制，RecyclerView则采用四级缓存机制，其中有一级缓存是开放给外部自定义的，因此大大提高了可扩展性。撇开这一层开放的缓存，RecyclerView的缓存优势主要在于从二级变到三级后，多出了一级负责减少bindView的调用。

另一方面，RecyclerView通过给每个Item增加flag的方式，在每次数据源变化时只更新需要更新的item，同样是减少bindView的调用。

##### 参考
- [Android ListView 与 RecyclerView 对比浅析--缓存机制](http://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653578065&idx=2&sn=25e64a8bb7b5934cf0ce2e49549a80d6&chksm=84b3b156b3c43840061c28869671da915a25cf3be54891f040a3532e1bb17f9d32e244b79e3f&scene=21#wechat_redirect)
- [RecyclerView必知必会](http://mp.weixin.qq.com/s/CzrKotyupXbYY6EY2HP_dA)

# Android体系ID (20170119)
* IMEI: Android手机的deviceId，需要权限
* 无线网卡地址: 
* ANDROID_ID: 每次恢复出厂设置生成的id
* 设备序列号（SN）: Build.SERIAL
* IDFA:
* OpenUDID:

# Kotlin (20170118)
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

