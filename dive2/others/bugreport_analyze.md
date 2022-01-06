# 简介

bugreport是Android的日志导出系统，有很多日志信息。OEM厂商报的bug基本都有完善的bugreport日志。

# 拉取方式

平时开发，只需要通过adb拉取即可。

而对于非开发，每个厂商也有拉取方式，每个厂商不一样。

小米（别名：284日志）

比较方便，不需要开启开发者模式或者usb调试，任何用户可以拉取（只要同意隐私弹框提示就行）。

方法：在手机的拨号界面输入“ *#*#284#*#* ”,系统会自动提示：“正在生成bug报告，大概需要1分钟”。

参考文档：https://dev.mi.com/docs/appsmarket/technical_docs/attain_bug_report/#bugqa

本地拉取bug报告：
```
//7.0和7.0以后
$ adb bugreport bugreport.zip
//6.0和6.0之前:
$ adb bugreport > bugreport.txt
```

# Bugreport的分析

## 官方文档

https://developer.android.com/studio/debug/bug-report?hl=zh-cn

https://source.android.com/source/read-bug-reports.html?hl=zh-cn

外部开源工具ChkBugReport：

https://www.jianshu.com/p/9c4a8642ccbf
（我看了一下，可以将bugreport进行拆分显示，但是实际不是特别有用）

## BugReport的组成

### 文件夹结构

下面是一个bugreport的文件结构，其中，我们一般关注的是三个部分：

* tombstones文件夹：查看哪些进程产生了native崩溃
* anr文件夹：查看哪些进程产生了ANR
* bugreport主文件：包含logcat、系统状态等主要信息

[!bugreport组成](./pic/bugreport_analyze_1.png)


### logcat日志
logcat的日志记录在 beginning of system 和 beginning of main 分割线之下。其中，system部分专门用于记录框架/系统方面的信息，main部分为"主要内容"区域，system记录的整体的时间比main中的更加长。输入法应用自己的日志记录在main当中。
```
------ SYSTEM LOG (logcat -v threadtime -d *:v) ------
--------- beginning of system
10-20 09:17:42.199  1000  1556  7937 I Timeline: ....
...

--------- beginning of main
10-20 09:17:42.002 10162  4558  4624 D ImageWallpaperRenderer: OFFSET  x: 0.0  y: 0.0
...
```

### crash日志和进程被杀死

由于bugreport中并不会带我们的CrashLog，因此要查看输入法或者其他进程是否崩溃，需要查看crash日志。通过 beginning of crash 分割线查看（注：QA的CI看崩溃也是这样看的）。
```
--------- beginning of crash
```

还有一种进程退出的方式是自杀或者被杀死。进程挂掉需要搜索的关键词是 has died。(has died的原因分析，同学可以进一步补充一下）

```
2021-12-07 19:51:53.875 1934-2855/? I/ActivityManager: Process com.xx.xxx (pid 20413) has died: fg  SVC 
```

### ANR日志

前面已经说了anr日志在单独的anr文件夹里面，但是logcat其实会输出多一点的信息，包括上下文情况，和Reason。所以在bugreport里面，通过如下两个tag再搜索一些信息来分析问题。
* 搜索am_anr，看发生anr的进程的概览
* 搜索ANR in，查看anr发生时候的logcat输出

```
grep "am_anr" bugreport-2015-10-01-18-13-48.txt
10-01 18:12:49.599  4600  4614 I am_anr  : [0,29761,com.google.android.youtube,953695941,executing service com.google.android.youtube/com.google.android.apps.youtube.app.offline.transfer.OfflineTransferService]
10-01 18:14:10.211  4600  4614 I am_anr  : [0,30363,com.google.android.apps.plus,953728580,executing service com.google.android.apps.plus/com.google.android.apps.photos.service.PhotosService]

grep "ANR in" bugreport-2015-10-01-18-13-48.txt
10-01 18:13:11.984  4600  4614 E ActivityManager: ANR in com.google.android.youtube
10-01 18:14:31.720  4600  4614 E ActivityManager: ANR in com.google.android.apps.plus
10-01 18:14:31.720  4600  4614 E ActivityManager: PID: 30363
10-01 18:14:31.720  4600  4614 E ActivityManager: Reason: executing service com.google.android.apps.plus/com.google.android.apps.photos.service.PhotosService
10-01 18:14:31.720  4600  4614 E ActivityManager: Load: 35.27 / 23.9 / 16.18
10-01 18:14:31.720  4600  4614 E ActivityManager: CPU usage from 16ms to 21868ms later:
10-01 18:14:31.720  4600  4614 E ActivityManager:   74% 3361/mm-qcamera-daemon: 62% user + 12% kernel / faults: 15276 minor 10 major
10-01 18:14:31.720  4600  4614 E ActivityManager:   41% 4600/system_server: 18% user + 23% kernel / faults: 18597 minor 309 major
10-01 18:14:31.720  4600  4614 E ActivityManager:   32% 27420/com.google.android.GoogleCamera: 24% user + 7.8% kernel / faults: 48374 minor 338 major
10-01 18:14:31.720  4600  4614 E ActivityManager:   16% 130/kswapd0: 0% user + 16% kernel
10-01 18:14:31.720  4600  4614 E ActivityManager:   15% 283/mmcqd/0: 0% user + 15% kernel
...
10-01 18:14:31.720  4600  4614 E ActivityManager:   0.1% 27248/irq/503-synapti: 0%
10-01 18:14:31.721  4600  4614 I ActivityManager: Killing 30363:com.google.android.apps.plus/u0a206 (adj 0): bg anr
```

### 进程和Activity的启动
* 使用 Start proc 关键词搜索进程的启动事件
* 使用 am_focused_activity 关键词搜索Activity到前台
* 使用 am_proc_died 和 am_proc_start 关键词搜索进程的启动和退出，短时间内大量进程死亡又再生，表示可能发生了系统颠簸。

### 内存
bugreport中也有不少内存相关的信息（但是目前来说，我还没有在查问题中用到过，有相关case的同学可以补充）。
* 使用 am_low_memory 关键词搜索是否系统存在low mem的情况。am_low_memory 条目表示最后一个缓存的进程已终止。在此之后，系统开始终止各项服务。
* 使用 am_meminfo 或者 am_pss 关键词搜索应用的内存情况（am_xx关键词里面有很多信息，可以对应到AMS源码中的一些逻辑，相关内容可以查下文列出的参考资料后再补充）
* 使用 dumpsys meminfo 或者 MEMINFO in pid 关键词，可以搜索到一些应用的完整meminfo的信息（包括dump的时间）

参考资料：eventLog的分析方法：http://gityuan.com/2016/05/15/event-log/

```
-------------------------------------------------------------------------------
DUMP OF SERVICE HIGH meminfo:
Applications Memory Usage (in Kilobytes):
Uptime: 25506626 Realtime: 151118191

// 中间有很多应用的内存dump情况
** MEMINFO in pid 7031 [com.miui.powerkeeper] **
                   Pss      Pss   Shared  Private   Shared  Private  SwapPss      Rss     Heap     Heap     Heap
                 Total    Clean    Dirty    Dirty    Clean    Clean    Dirty    Total     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------   ------   ------   ------   ------
  Native Heap     6888        0      940     6880       40        0     3758     7860    22172    19479     2692
  Dalvik Heap     4995        0      836     4976      712        0      640     6524    11412     5706     5706
  ...

--------- 30.003s was the duration of dumpsys meminfo, ending at: 2021-10-20 09:19:57
-------------------------------------------------------------------------------
```

### 耗电
bugreport里面本身有比较完整的battery stats的信息，不过可读性比较差，有兴趣的同学可以继续研究和补充。
使用Battery Historian工具，解析我们导出的bugreport，可以进行完整的电量分析。
参考官方文档：[使用 Battery Historian 分析耗电情况](https://developer.android.google.cn/topic/performance/power/battery-historian?hl=zh-cn)

不过Battery Historian这个工具的安装实在是非常麻烦，我自己试了好久，都由于各种网络问题没有尝试成功😭。
推荐这篇博文：[Mac 中 Battery Historain 安装与使用](https://www.cnblogs.com/xiaxveliang/p/14869914.html) 里面记录了很多问题和解决方法，有兴趣的同学可以尝试下。
终极方法：有人已经搭好了一个线上环境，直接使用就好了：https://bathist.ef.lc/

工具会展示一些数据和图表，大概长这样：

[!耗电分析图片](./pic/bugreport_analyze_2.png)

### 其他
官方文档中还有点内容，了解的同学可以补充
