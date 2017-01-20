# Android的Resource资源概览
## res/文件夹目录结构和使用概览
![res/文件夹目录结构](http://bos.nj.bpc.baidu.com/v1/agroup/8b5e5d7cc3b52d1c4b0ca1c53448d7caea9bd3b2)

## Resources相关类的关系
![图片](http://bos.nj.bpc.baidu.com/v1/agroup/9d915a7dc6ee6cd547815ee5d2db43e147bbc2cb)
* R
主要管理aapt分配出的id

* Resources
负责根据R文件中编译好的id，获取资源中的值

* AttributeSet
主要是在View类的构造函数中见到，这是一个接口，代表了一个xml文件所有的属性本身和属性值的集合。提供了操作属性的方法（属性其实就是定义在<attr>标签中的内容，我们常见的anddroid:layout_width其实就是一个<attr>。[ ] 存疑）。提供了获取属性名、获取属性值、解析属性值所对应的实际资源等功能。

* TypedArray
对AttributeSet分组后，对应于某个Stylable的集合。即:一组<attr>的封装。我们可以通过context.obtainStyledAttributes(R.stylable.xxx, …)来获取该对象，从而更加方便地获取我们需要的属性对。另外还提供了一些直接取值的方法：getColor,getFloat等，无需再解析string类型的值。当然，系统在解析xml的时候（例如解析LayoutParams的时候），实际上也是利用了TypedArray

* AssetManager
真正管理资源的类。包括assert/和res/下的资源管理。在jni层面实现根据R文件中的id获取资源的功能。

# Resources对象的获取
入口：
Context.getResources()
要点：

- 进入ActivityThread里面获取资源
- 资源的ResourceKey和apk的路径强相关
- Resource实例的创建依赖于AssetManager


### 从Context获取
获取过程：
![图片](http://bos.nj.bpc.baidu.com/v1/agroup/879d2f2a344e250f1700c93632e2035f6396bbae)
具体看代码：
```
/** LoadedApk.java */
public Resources getResources(ActivityThread mainThread) {
    if (mResources == null) {
        mResources = mainThread.getTopLevelResources(mResDir, mSplitResDirs, mOverlayDirs,
                mApplicationInfo.sharedLibraryFiles, Display.DEFAULT_DISPLAY, null, this);
    }
    return mResources;
}
```
首先看到，在LoadedApk当中调用了mainThread.getTopLevelResources，并传入了几个路径：mResDir, mSplitResDirs, mOverlayDirs。
```
/**
 * ActivityThread.java
 * Creates the top level resources for the given package.
 */
Resources getTopLevelResources(String resDir, String[] splitResDirs, String[] overlayDirs,
        String[] libDirs, int displayId, Configuration overrideConfiguration,
        LoadedApk pkgInfo) {
    return mResourcesManager.getTopLevelResources(resDir, splitResDirs, overlayDirs, libDirs,
            displayId, overrideConfiguration, pkgInfo.getCompatibilityInfo());
}
```
再进入ResourcesManager.java。
这个方法主要做了这几件事情：

* 构建一个ResourcesKeys: 主要包含resDir, displayId, overrideConfigCopy, scale几个参数
* 从``mActivieResources:Map<ResourcesKey, WeakReference<Resources>>``里面查询是否已经有对应的Resources了。
* 如果没有，构建AssetManager对象，然后依次调用addAssetPath添加路径： 
		* resDir
		* splitResDirs
		* overlayDirs
		* libDirs中.apk后缀的路径
* 构建Resources对象
* 添加到Map中，再返回
```
/**
* Creates the top level Resources for applications with the given compatibility info.
*
* @param resDir the resource directory.
* @param splitResDirs split resource directories.
* @param overlayDirs the resource overlay directories.
* @param libDirs the shared library resource dirs this app references.
* @param displayId display Id.
* @param overrideConfiguration override configurations.
* @param compatInfo the compatibility info. Must not be null.
*/
Resources getTopLevelResources(String resDir, String[] splitResDirs,
        String[] overlayDirs, String[] libDirs, int displayId,
        Configuration overrideConfiguration, CompatibilityInfo compatInfo) {
    ...
    ResourcesKey key = new ResourcesKey(resDir, displayId, overrideConfigCopy, scale);
    Resources r;
    synchronized (this) {
        
        WeakReference<Resources> wr = mActiveResources.get(key);
        r = wr != null ? wr.get() : null;
        if (r != null && r.getAssets().isUpToDate()) {
            return r;
        }
    }

    AssetManager assets = new AssetManager();
    if (resDir != null) {
        if (assets.addAssetPath(resDir) == 0) {
            return null;
        }
    }

    if (splitResDirs != null) {
        for (String splitResDir : splitResDirs) {
            if (assets.addAssetPath(splitResDir) == 0) {
                return null;
            }
        }
    }

    if (overlayDirs != null) {
        for (String idmapPath : overlayDirs) {
            assets.addOverlayPath(idmapPath);
        }
    }

    if (libDirs != null) {
        for (String libDir : libDirs) {
            if (libDir.endsWith(".apk")) {
                if (assets.addAssetPath(libDir) == 0) {
                    Log.w(TAG, "Asset path '" + libDir +
                            "' does not exist or contains no resources.");
                }
            }
        }
    }

    ... // 这里获取一个Configuration对象
    r = new Resources(assets, dm, config, compatInfo);

    synchronized (this) {
        WeakReference<Resources> wr = mActiveResources.get(key);
        Resources existing = wr != null ? wr.get() : null;
        if (existing != null && existing.getAssets().isUpToDate()) {
            r.getAssets().close();
            return existing;
        }

        mActiveResources.put(key, new WeakReference<>(r));
        return r;
    }
}
```


这里要特别介绍一下AssetManager的创建：
```
public AssetManager() {
    synchronized (this) {
        init(false); // native方法
        ensureSystemAssets(); // native方法
    }
}
```
其中，init方法在native层，会添加系统资源路径（保证当前程序也可以访问系统资源）。其中，系统资源路径拼接后是：
/system/framework/framework-res.apk

### 从PackageManager获取
从PackageManager获取Resources，主要作用是可以访问其他程序中的public资源。具体来讲，代码是这样的：
```
PackageManager pm = mContext.getPackageManager();
pm.getResourcesForApplication("com.android.xxx");
首先，mContext.getPackageManager获取到的，是一个ApplicationPackageManager的实例。ApplicationPackageManager.getResourcesForApplication()方法，依旧是调用了getTopLevelResources方法。
/** ApplicationPackageManager.java */
public Resources getResourcesForApplication(@NonNull ApplicationInfo app)
        throws NameNotFoundException {
    if (app.packageName.equals("system")) {
        return mContext.mMainThread.getSystemContext().getResources();
    }
    final boolean sameUid = (app.uid == Process.myUid());
    final Resources r = mContext.mMainThread.getTopLevelResources(
            sameUid ? app.sourceDir : app.publicSourceDir,
            sameUid ? app.splitSourceDirs : app.splitPublicSourceDirs,
            app.resourceDirs, app.sharedLibraryFiles, Display.DEFAULT_DISPLAY,
            null, mContext.mPackageInfo);
    if (r != null) {
        return r;
    }
    throw new NameNotFoundException("Unable to open " + app.publicSourceDir);
}
```
# aapt工具简介
TODO: TODO: 

# 实践：“插件化”方式实现应用程序主题切换
参考：
https://my.oschina.net/ososchina/blog/668384
https://my.oschina.net/ososchina/blog/396173
关键代码：
```
String dexPath_tmp=params[0]; // 指定apk的路径
PackageManager mPm=mContext.getPackageManager();            
                                  
AssetManager assetManager = AssetManager.class.newInstance();            
Method addAssetPath = assetManager.getClass().getMethod("addAssetPath", String.class);           
addAssetPath.invoke(assetManager, dexPath_tmp); // 反射构建AssetManager，调用addAssetPath    
                     
Resources superRes = mContext.getResources();            
Resources skinResource=new Resources(assetManager, superRes.getDisplayMetrics(), superRes.getConfiguration()); // 构建Resources对象                    
return skinResource;
```


# 参考：
[ ] https://zhuanlan.zhihu.com/p/22864155
[ ] http://www.maplejaw.com/2016/05/26/Android%E6%8F%92%E4%BB%B6%E5%8C%96%E6%8E%A2%E7%B4%A2%EF%BC%88%E4%BA%8C%EF%BC%89%E8%B5%84%E6%BA%90%E5%8A%A0%E8%BD%BD/

