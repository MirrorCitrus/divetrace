# Systrace的使用和分析

## 使用

### 环境条件

- Android SDK Tools 20
- 安装的python并添加了环境变量
- Android 4.1以上的手机

### 启动trace并记录

- Tools -> Android -> Android Device Monitor
- 选择app，点击Systrace按钮
- 设置保存的文件路径、监控的时间、监控的内容等，开始

## Systrace的分析

### 分析帧率

基本操作：

- 'w'按键：放大
- 's'按键：缩小
- 'd'按键：向右平移
- 'a'按键：向左平移

首先在左边纵向的pid列表中，定位到当前应用的包名所在的进程项，展开后包含`deliverInputEvent`, `UI Thread`, `RenderThread`三个子项。我们重点分析这部分内容。

右侧有一行写有F的圈圈，表示每个Frame的综合评价。如果是绿色，表示这一帧的渲染是ok的。如果是红色或者黄色，表示渲染超时了。
点击这个F圆圈，在下部的窗口中会展示一些详细信息。如果这一帧的渲染没有任何问题，下面的内容是空的；如果这一帧有问题，会展示Alert信息（我们稍后详细讨论Alert）。

![Systrace的界面概览和F标记](/assets/systrace1.png)
> 图：Systrace的界面概览和F标记

![观察有性能问题的Frame的提示信息](/assets/systrace2.png)
> 图：观察有性能问题的Frame的提示信息 

点击其中的一个方块，可以使这一段耗时高亮，在下面会展示出详细的耗时信息。

![详细](/assets/systrace3.png)
> 图：选中的Slice的详细耗时信息

### 举例：一个Frame的耗时信息详解

一帧的绘制大概需要经过这样几个区间（纵向的灰色竖条表示的是Vsync的时间）。

- Choreographer#doFrame （UI Thread）
    - traversal
        - measure（如果measure了多次，会在trace中有体现）
        - layout
        - draw
- DrawFrame (RenderThread)
    - syncFrameState
        - uploadTexture
    - flushDrawingCommands
    - eglSwapBuffersWithDamageKHR

![](/assets/systrace2.png)
> TODO: 添加图示，并附上部分解释

## 分析Alert的信息

Systrace会对trace中的一些事件执行自动分析，然后将一些性能问题以Alert形式标出，并给出一定的建议。前面已经提到，在有性能问题的Frame上点击，会在下方的详情中展示Alert，点击Alert项，会给出详细的耗时信息、相关描述，有的还会给出链接，链到指定的trace块中。

![image](http://note.youdao.com/favicon.ico)

> TODO 插图

另外，右侧有一个Alert的tab页，会把此次trace中所有的Alert信息列举出来。点击其中一个Alert，对应的F圈会高亮，从而可以定位到有问题性能的帧。

发生Alert之后，可能我们已有的信息还不足以解决这些性能问题。此时，我们可以考虑先使用TraceView来定位耗时较长的方法，然后针对这些有问题的方法，在代码中添加Mark后，再执行Systrace进行确认和进一步的分析。（注意：TraceView本身有一定的性能损耗，会导致得出的结果和实际有较大出入，所以不建议使用TraceView取代Systrace，而是使用它作为查找问题的一个辅助工具）。

下一节我们进一步介绍，如何在代码中添加Systrace的Mark。

### 举例：一些Alert信息的例子

## 从代码中追踪性能问题

从Android 4.3（API 18）开始，使用`Trace`类，可以在代码中添加更加详细的标记。使用`Trace.beginSection(String)`开始一段追踪，使用`Trace.endSection()`结束最近的一次追踪。

下面是一段示例代码：

```
public class MyAdapter extends RecyclerView.Adapter<MyViewHolder> {

    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        Trace.beginSection("MyAdapter.onCreateViewHolder");
        MyViewHolder myViewHolder;
        try {
            myViewHolder = MyViewHolder.newInstance(parent);
        } finally {
            Trace.endSection();
        }
        return myViewHolder;
    }

   @Override
    public void onBindViewHolder(MyViewHolder holder, int position) {
        Trace.beginSection("MyAdapter.onBindViewHolder");
        try {
            try {
                Trace.beginSection("MyAdapter.queryDatabase");
                RowItem rowItem = queryDatabase(position);
                mDataset.add(rowItem);
            } finally {
                Trace.endSection();
            }
            holder.bind(mDataset.get(position));
        } finally {
            Trace.endSection();
        }
    }
}
```




## References

- [x] [性能工具Systrace的使用](http://gityuan.com/2016/01/17/systrace/)
- [x] [Analyzing UI Performance with Systrace](https://developer.android.com/studio/profile/systrace.html#app-trace)