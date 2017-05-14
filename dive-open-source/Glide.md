# 简介

# WalkThrough

#### 快速使用：

```
Glide.with(this).load(url).into(imageView);
```
#### API详解

- 基础三步
    - `Glide.with():RequestManager` : 传入不同的context创建图片加载实例
        - Activity/Fragment/Application的context，内部根据具体context的生命周期处理加载的策略 
    - `load():DrawableTypeRequest<T>` : 加载图片
        - 加载本地图片: load(imgFile)
        - 加载应用程序资源: load(R.drawable.xxx)
        - 加载二进制流: load(imgByteArray)
        - 加载uri: load(imgUri)
    - `into():Target<T>` : 指定展示目标
        - into(imageView)
- 占位图
    - `placeHolder(R.drawable.xxx):DrawableRequestBuilder<T>`
    - `error(R.drawable.xxx):DrawableRequestBuilder<T>`
- 缓存设置
    - `diskCacheStrategy(...)`: 设置缓存策略
        - `Strategy.ALL` :
        - `Strategy.NONE` :
        - `Strategy.SOURCE` :
        - `Strategy.RESULT` :
- 指定图片格式
    - `asBitmap()`
    - `asGif()`
- 指定尺寸
    - 默认自适应ImageView的大小
    - `override(100,100)`

# 源码解析

## with()

`with()`方法是Glide的静态方法，接收一个Context对象，创建一个RequestManager对象返回。with方法提供了多个方法重载，根据传入的Context的不同，构建不同的生命周期：

```
public class Glide {

    public static RequestManager with(Context context) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(context);
    }

    public static RequestManager with(Activity activity) { }

    public static RequestManager with(FragmentActivity activity) { }

    @TargetApi(Build.VERSION_CODES.HONEYCOMB)
    public static RequestManager with(android.app.Fragment fragment) { }

    public static RequestManager with(Fragment fragment) { }
}
```

其中，`RequestManagerRetriever.get()`是一个单例的实现；下面来看`retriever.get()`的方法：

```
public class RequestManagerRetriever implements Handler.Callback {

    public RequestManager get(Context context) {
        if (context == null) {
            throw new IllegalArgumentException("You cannot start a load on a null Context");
        } else if (Util.isOnMainThread() && !(context instanceof Application)) {
            if (context instanceof FragmentActivity) {
                return get((FragmentActivity) context);
            } else if (context instanceof Activity) {
                return get((Activity) context);
            } else if (context instanceof ContextWrapper) {
                return get(((ContextWrapper) context).getBaseContext());
            }
        }
    
        return getApplicationManager(context);
    }
    
    private RequestManager getApplicationManager(Context context) {
        // Either an application context or we're on a background thread.
        if (applicationManager == null) {
            synchronized (this) {
                if (applicationManager == null) {
                    // Normally pause/resume is taken care of by the fragment we add to the fragment or activity.
                    // However, in this case since the manager attached to the application will not receive lifecycle
                    // events, we must force the manager to start resumed using ApplicationLifecycle.
                    applicationManager = new RequestManager(context.getApplicationContext(),
                            new ApplicationLifecycle(), new EmptyRequestManagerTreeNode());
                }
            }
        }
    
        return applicationManager;
    }
}
```

可见，如果是主线程，get方法会根据context是否是FragmentActivity/Activity/ContextWrapper进行不同的处理；其余情况，都走getApplicationManager，这个也是最简单的情况，获取到的`applicationManager`实例，则是一个单例。

接着看FragmentActivity/Activity/Fragment的`ger`:

```
    public RequestManager get(FragmentActivity activity) {
        if (Util.isOnBackgroundThread()) {
            return get(activity.getApplicationContext());
        } else {
            assertNotDestroyed(activity);
            FragmentManager fm = activity.getSupportFragmentManager();
            return supportFragmentGet(activity, fm);
        }
    }

    public RequestManager get(Fragment fragment) {
        // ...
        FragmentManager fm = fragment.getChildFragmentManager();
        return supportFragmentGet(fragment.getActivity(), fm);
    }
    
    @TargetApi(Build.VERSION_CODES.HONEYCOMB)
    public RequestManager get(Activity activity) {
        // ...
        android.app.FragmentManager fm = activity.getFragmentManager();
        return fragmentGet(activity, fm);
    }
    
    @TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR1)
    public RequestManager get(android.app.Fragment fragment) {
        // ...
        android.app.FragmentManager fm = fragment.getChildFragmentManager();
        return fragmentGet(fragment.getActivity(), fm);
    }
    
```

可以看到，Glide需要获取传入的Context对应的FragmentManager对象，再通过supportFragmentGet/fragmentGet方法获取RequestManager对象。

```
    @TargetApi(Build.VERSION_CODES.HONEYCOMB)
    RequestManager fragmentGet(Context context, android.app.FragmentManager fm) {
        RequestManagerFragment current = getRequestManagerFragment(fm);
        RequestManager requestManager = current.getRequestManager();
        if (requestManager == null) {
            requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
            current.setRequestManager(requestManager);
        }
        return requestManager;
    }
    
    @TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR1)
    RequestManagerFragment getRequestManagerFragment(final android.app.FragmentManager fm) {
        RequestManagerFragment current = (RequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
        if (current == null) {
            current = pendingRequestManagerFragments.get(fm);
            if (current == null) {
                current = new RequestManagerFragment();
                pendingRequestManagerFragments.put(fm, current);
                fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
                handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();
            }
        }
        return current;
    }
```

supportFragmentGet/fragmentGet方法内部的主要任务是：向当前的Activity当中添加一个隐藏的Fragment。

// TODO

## load()

我们以`load(String)`为例，看整体的加载流程：

```
public class RequestManager implements LifecycleListener {

    private final OptionsApplier optionsApplier;

    public DrawableTypeRequest<String> load(String string) {
        return (DrawableTypeRequest<String>) fromString().load(string);
    }

    public DrawableTypeRequest<String> fromString() {
        return loadGeneric(String.class);
    }

    private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
        ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
        ModelLoader<T, ParcelFileDescriptor> fileDescriptorModelLoader =
                Glide.buildFileDescriptorModelLoader(modelClass, context);
        
        return optionsApplier.apply(
                new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                        glide, requestTracker, lifecycle, optionsApplier));
    }

    ...

}
```
可见，最终都是调用了`loadGeneric()`方法，根据不同的class类型，返回`DrawableTypeRequest<T>`的实例。这个类的最父类是`GenericRequestBuilder`, 这是Glide中进行图片加载和加载设置的基类。Glide的大多数的设置API其实就是这个类的API。而整个继承树也是比较庞大：

![Glide的requestBuilder系统类图](/assets/Glide的requestBuilder系统类图.png)

得到`DrawableTypeRequest`的实例后，load方法其实尤其简单：

```
public class GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> {

   /**
     * Sets the specific model to load data for.
     * <p>
     *      This method must be called at least once before {@link #into(com.bumptech.glide.request.target.Target)} is
     *      called.
     * </p>
     * @param model The model to load data for, or null.
     * @return This request builder.
     */
    public GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> load(ModelType model) {
        this.model = model;
        isModelSet = true;
        return this;
    }
}
```

## into()

into方法的实现在`GenericRequestBuilder`中，这个方法首先根据当前的ImageView，和当前request的transcodeClass构建一个Target对象，然后调用重载方法`into(Y extends Target<Transcode>)`。而重载的into方法，首先会将传入Target所对应的之前的加载请求清除并释放相关资源；然后执行此次的request，同时此次的request还会作为lifecycle的监听器，监听总的生命周期的变化。

```
public class GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> {

    protected final RequestTracker requestTracker;
    protected final Lifecycle lifecycle;
    
     /**
     * Sets the {@link ImageView} the resource will be loaded into, cancels any existing loads into the view, and frees
     * any resources Glide may have previously loaded into the view so they may be reused.
     *
     * @see Glide#clear(android.view.View)
     *
     * @param view The view to cancel previous loads for and load the new resource into.
     * @return The {@link com.bumptech.glide.request.target.Target} used to wrap the given {@link ImageView}.
     */
    public Target<TranscodeType> into(ImageView view) {
        // ...
        return into(glide.buildImageViewTarget(view, transcodeClass));
    }
    
}

```

再看一下`Glide.buildImageViewTarget`的实现：

```
public class Glide {

    private final ImageViewTargetFactory imageViewTargetFactory = new ImageViewTargetFactory();

    <R> Target<R> buildImageViewTarget(ImageView imageView, Class<R> transcodedClass) {
        return imageViewTargetFactory.buildTarget(imageView, transcodedClass);
    }
}

public class ImageViewTargetFactory {

    @SuppressWarnings("unchecked")
    public <Z> Target<Z> buildTarget(ImageView view, Class<Z> clazz) {
        if (GlideDrawable.class.isAssignableFrom(clazz)) {
            return (Target<Z>) new GlideDrawableImageViewTarget(view);
        } else if (Bitmap.class.equals(clazz)) {
            return (Target<Z>) new BitmapImageViewTarget(view);
        } else if (Drawable.class.isAssignableFrom(clazz)) {
            return (Target<Z>) new DrawableImageViewTarget(view);
        } else {
            throw new IllegalArgumentException("Unhandled class: " + clazz
                    + ", try .as*(Class).transcode(ResourceTranscoder)");
        }
    }
}
```

最后看重载的into方法：

```
public class GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> {

    protected final RequestTracker requestTracker;
    protected final Lifecycle lifecycle;
    
    /**
     * Set the target the resource will be loaded into.
     *
     * @see Glide#clear(com.bumptech.glide.request.target.Target)
     *
     * @param target The target to load the resource into.
     * @return The given target.
     */
    public <Y extends Target<TranscodeType>> Y into(Y target) {
        // ...

        Request previous = target.getRequest();

        if (previous != null) {
            previous.clear();
            requestTracker.removeRequest(previous);
            previous.recycle();
        }

        Request request = buildRequest(target);
        target.setRequest(request);
        lifecycle.addListener(target);
        requestTracker.runRequest(request);

        return target;
    }
}
```

过程比较清晰。不过其中，真正封装了加载图片的类是Request，我们看看buildRequest的实现：
```
private Request buildRequest(Target<TranscodeType> target) {
    if (priority == null) {
        priority = Priority.NORMAL;
    }
    return buildRequestRecursive(target, null);
}

private Request buildRequestRecursive(Target<TranscodeType> target, ThumbnailRequestCoordinator parentCoordinator) {
    if (thumbnailRequestBuilder != null) {
        
        // Recursive case: contains a potentially recursive thumbnail request builder.
        // ...

        ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
        Request fullRequest = obtainRequest(target, sizeMultiplier, priority, coordinator);
        // Guard against infinite recursion.
        isThumbnailBuilt = true;
        // Recursively generate thumbnail requests.
        Request thumbRequest = thumbnailRequestBuilder.buildRequestRecursive(target, coordinator);
        isThumbnailBuilt = false;
        coordinator.setRequests(fullRequest, thumbRequest);
        return coordinator;
    } else if (thumbSizeMultiplier != null) {
        // Base case: thumbnail multiplier generates a thumbnail request, but cannot recurse.
        ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
        Request fullRequest = obtainRequest(target, sizeMultiplier, priority, coordinator);
        Request thumbnailRequest = obtainRequest(target, thumbSizeMultiplier, getThumbnailPriority(), coordinator);
        coordinator.setRequests(fullRequest, thumbnailRequest);
        return coordinator;
    } else {
        // Base case: no thumbnail.
        return obtainRequest(target, sizeMultiplier, priority, parentCoordinator);
    }
}

private Request obtainRequest(Target<TranscodeType> target, float sizeMultiplier, Priority priority,
        RequestCoordinator requestCoordinator) {
    return GenericRequest.obtain(
            loadProvider,
            model,
            signature,
            context,
            priority,
            target,
            sizeMultiplier,
            placeholderDrawable,
            placeholderId,
            errorPlaceholder,
            errorId,
            fallbackDrawable,
            fallbackResource,
            requestListener,
            requestCoordinator,
            glide.getEngine(),
            transformation,
            transcodeClass,
            isCacheable,
            animationFactory,
            overrideWidth,
            overrideHeight,
            diskCacheStrategy);
}
```

buildRequest方法处理了递归式缩略图处理（渐进式加载？），除去这些操作，核心的调用就是`obtainRequest`，这个方法内部，又调用了`GenericRequest.obtain`，把之前设置的各种参数全部传递进去。从方法名我们很容易明白：方法内部其实核心是new了一个Request，并将各种参数设置给它作为成员变量。

最后我们看runRequest方法，核心就是调用了request.begin方法，非常简单。

```
/**
 * Starts tracking the given request.
 */
public void runRequest(Request request) {
    requests.add(request);
    if (!isPaused) {
        request.begin();
    } else {
        pendingRequests.add(request);
    }
}
```

## Request.begin

其实到前面为止，外部的调用已经不需要再继续关心如何加载图片了。这一小节我们再仔细看看一个Request是如何具体地加载图片的。

```
public final class GenericRequest<A, T, Z, R> implements Request, SizeReadyCallback,
        ResourceCallback {
    
    @Override
    public void begin() {
        if (model == null) {
            onException(null); // 这里会设置errorPlaceHolder
            return;
        }

        status = Status.WAITING_FOR_SIZE;
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
            onSizeReady(overrideWidth, overrideHeight);
        } else {
            target.getSize(this);
        }

        if (!isComplete() && !isFailed() && canNotifyStatusChanged()) {
            target.onLoadStarted(getPlaceholderDrawable()); // 通知target： load started，target在这里设置loading占位图
        }
        
    }
}
```

这个方法会根据情况决定是回调target的onLoadStarted还是onLoadError，通知target进行占位图的设置。中间会根据外部是否设置了size来决定是直接调用onSizeReady还是调用target.getSize(SizeReadyCallback)。第二种情况，Glide要求展示图片的Target实现getSize接口，在外部未设置具体大小时进行默认的测量，测量结束后，调用传入的SizeReadyCallback.onSizeReady()方法。总之，大小确定后，都调用onSizeReady:

```
public final class GenericRequest<A, T, Z, R> implements Request, SizeReadyCallback,
        ResourceCallback {
    
    private LoadProvider<A, T, Z, R> loadProvider;
    private Engine engine;
    
    @Override
    public void onSizeReady(int width, int height) {
        
        if (status != Status.WAITING_FOR_SIZE) {
            return;
        }
        status = Status.RUNNING;

        width = Math.round(sizeMultiplier * width);
        height = Math.round(sizeMultiplier * height);

        ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
        final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);

        ResourceTranscoder<Z, R> transcoder = loadProvider.getTranscoder();
        
        loadedFromMemoryCache = true;
        loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,
                priority, isMemoryCacheable, diskCacheStrategy, this);
        loadedFromMemoryCache = resource != null;
        
    }
}

```

这里的核心是要调用engine.load方法，而这个方法的相关参数，则是需要从成员变量`loadProvider`获得。LoadProvider是一个接口，表征了数据的编解码关系，这个类的构建其实是在`DrawableTypeRequest.buildProvider(...) : FixedLoadProvider<A, ImageVideoWrapper, Z, R>`，这里的详细不再细说。

我们再看engine的构造。这个engine其实是引用了Glide的成员变量，构造如下：

```
public class GlideBuilder {

    Glide createGlide() {
        if (sourceService == null) {
            final int cores = Math.max(1, Runtime.getRuntime().availableProcessors());
            sourceService = new FifoPriorityThreadPoolExecutor(cores);
        }
        if (diskCacheService == null) {
            diskCacheService = new FifoPriorityThreadPoolExecutor(1);
        }

        MemorySizeCalculator calculator = new MemorySizeCalculator(context);
        if (bitmapPool == null) {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
                int size = calculator.getBitmapPoolSize();
                bitmapPool = new LruBitmapPool(size);
            } else {
                bitmapPool = new BitmapPoolAdapter();
            }
        }

        if (memoryCache == null) {
            memoryCache = new LruResourceCache(calculator.getMemoryCacheSize());
        }

        if (diskCacheFactory == null) {
            diskCacheFactory = new InternalCacheDiskCacheFactory(context);
        }

        if (engine == null) {
            engine = new Engine(memoryCache, diskCacheFactory, diskCacheService, sourceService);
        }

        if (decodeFormat == null) {
            decodeFormat = DecodeFormat.DEFAULT;
        }

        return new Glide(engine, memoryCache, bitmapPool, context, decodeFormat);
    }
}
```

可见，Engine主要负责启动图片的加载（包括网络访问、磁盘加载等）,管理内存中的资源和缓存的资源的。我们详细看它的load方法：

```
public class Engine implements EngineJobListener,
        MemoryCache.ResourceRemovedListener,
        EngineResource.ResourceListener {

    /**
     * Starts a load for the given arguments. Must be called on the main thread.
     *
     * <p>
     *     The flow for any request is as follows:
     *     <ul>
     *         <li>Check the memory cache and provide the cached resource if present</li>
     *         <li>Check the current set of actively used resources and return the active resource if present</li>
     *         <li>Check the current set of in progress loads and add the cb to the in progress load if present</li>
     *         <li>Start a new load</li>
     *     </ul>
     * </p>
     *
     * <p>
     *     Active resources are those that have been provided to at least one request and have not yet been released.
     *     Once all consumers of a resource have released that resource, the resource then goes to cache. If the
     *     resource is ever returned to a new consumer from cache, it is re-added to the active resources. If the
     *     resource is evicted from the cache, its resources are recycled and re-used if possible and the resource is
     *     discarded. There is no strict requirement that consumers release their resources so active resources are
     *     held weakly.
     * </p>
     * /
    public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {

        final String id = fetcher.getId();
        EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
                loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
                transcoder, loadProvider.getSourceEncoder());

        EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
        if (cached != null) {
            cb.onResourceReady(cached); // load from cache
            return null;
        }

        EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
        if (active != null) {
            cb.onResourceReady(active); // Loaded resource from active resources
            return null;
        }

        EngineJob current = jobs.get(key);
        if (current != null) {
            current.addCallback(cb); // Added to existing load
            return new LoadStatus(cb, current);
        }

        EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
        DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
                transcoder, diskCacheProvider, diskCacheStrategy, priority);
        EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
        jobs.put(key, engineJob);
        engineJob.addCallback(cb);
        engineJob.start(runnable); // start a new job
        
        return new LoadStatus(cb, engineJob);
    }

    ...
}
```

这个方法里清晰地展示了Glide的缓存处理策略，注释也非常清晰。而如果缓存不命中，则执行一个新的EnginRunnable。EngineJob其实是对一个加载的过程进行一些异步的管理，即:将EngineRunnable投递到一个线程池中去执行，这一块我们暂时不看。我们看EngineRunnable的工作：

- EngineRunnable.run()
    - EngineRunnable.decode()-> resource : Resource<?>
        - decodeFromSource() : Resouce<?>
            - DecodeJob.decodeFromSource()
                - fetcher.loadData() // 这里的fetcher是ImageVideoFetcher对象
                    - streamFetcher.loadData() -> is : InputStream // 这里的streamFetcher是HttpUrlFetcher对象，进行网络请求
                    - fileDescriptorFetcher.loadData()
                    - return new ImageVideoWrapper(is, fd)
                - decodeFromSourceData() -> decoded:Resource<T> // T的实际类型是GifBitmapWrapper
                    - if(diskCacheStrategy.cacheSource()) : cacheAndDecodeSourceData(data)
                    - else: loadProvider.getSourceDecoder().decode(data...) 
                        - GifBitmapWrapperResourceDecoder.decode()
                            - decodeStream() // 从网络的流中读取
                                - 读取到前几个字节信息，如果是gif: 则decodeGifWrapper()
                                - 如果是静态图，则decodeBitmapWrapper() // 内部含有downsample操作
                - fetcher.cleanup()
                - transformEncodeAndTranscode(decoded)->Resource<Z>
                    - transform(decoded)->transformed:Resource<T>
                    - writeTransformedToCache()
                    - transcoder.transcode(transformed)->Resource<Z> // transcoder是GifBitmapWrapperDrawableTranscoder对象，用于将gif->GifDrawable, Bitmap->GlideBitmapDrawable
                    - 最终返回的是Resource<GlideDrawable>
    - EngineRunnable.onLoadCompleted(resource)
        - manager.onResourceReady(resource) // manager是EngineJob对象 
        - listener.onResourceReady(key, null); // listener是GenericRequest对象
            - resource.get() -> received
            - target.onResourceReady(result)
            - notifyLoadSuccess()


# References

- [ ] [github](https://github.com/bumptech/glide)
- [x] [Android图片加载框架最全解析（一），Glide的基本用法](http://blog.csdn.net/guolin_blog/article/details/53759439)
- [x] [ Android图片加载框架最全解析（二），从源码的角度理解Glide的执行流程](http://blog.csdn.net/guolin_blog/article/details/53939176)
- [ ] [Google推荐的图片加载库Glide介绍](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0327/2650.html)