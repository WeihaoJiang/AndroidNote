---
title: Glide源码分析
date: 2019-10-24
tags: [Android]
categories: Android
---


## Glide加载流程？

先看一眼Glide最简单的使用
```
  Glide.with(this).load(url).into(imageView);
```
### 架构图
再看一眼整体架构图
![图片](https://s2.ax1x.com/2019/10/28/KcUoKH.png)
我们也从这三个方面来分析
### with()方法
首先with()方法的参数是什么？
- 1 传递Activity，则此次加载图片会在退出Activity后自动取消；
- 2 传递Fragment，则加载图片会在Fragment销毁时取消；
- 3 传递Application，那么Glide将无法管理图片请求生命周期。
然后看with()方法的源码：
```java
 public static RequestManager with(Context context) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(context);
    }

    public static RequestManager with(Activity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }

    public static RequestManager with(FragmentActivity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }

    @TargetApi(Build.VERSION_CODES.HONEYCOMB)
    public static RequestManager with(android.app.Fragment fragment) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(fragment);
    }

    public static RequestManager with(Fragment fragment) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(fragment);
    }

```
每一个with()方法重载的代码都非常简单，都是先调用RequestManagerRetriever的静态get()方法得到一个RequestManagerRetriever对象，这个静态get()方法就是一个单例实现。然后再调用RequestManagerRetriever的实例get()方法，去获取RequestManager对象。换句话说，每一个Activity存在一个对应的RequestManager，每一个不同的Fragment也有其对应的RequestManager，而整个应用运行阶段同样会有一个RequestManager。这些不同的RequestManager通过RequestManagerRetriever.get获取。RequestManagerRetriever同样重载了多个get方法，以get(FragmentActivity)为例:
```java
//RequestManagerRetriever
public RequestManager get(@NonNull FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      FragmentManager fm = activity.getSupportFragmentManager();
      return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
}
```
这里从是否在主线程的角度来看：
- 1 若在主线程，则获得Activity对应的FragmentManager来获得RequestManager（Fragment同理）。
- 2 若在非主线程，那么不管你是传入的Activity还是Fragment，都会被强制当成Application来处理 。

也可以从传入Application和非Application的角度来看：
- 1 传Application，因为Application对象的生命周期即应用程序的生命周期，因此Glide并不需要做什么特殊的处理，它自动就是和应用程序的生命周期是同步的，如果应用程序关闭的话，Glide的加载也会同时终止。
- 2 传非Application，传入非Application参数的情况。不管在Glide.with()方法中传入的是Activity、FragmentActivity、v4包下的Fragment、还是app包下的Fragment，最终的流程都是一样的，那就是会向当前的Activity当中添加一个隐藏的Fragment。因为Glide需要知道加载的生命周期。很简单的一个道理，如果你在某个Activity上正在加载着一张图片，结果图片还没加载出来，Activity就被用户关掉了，那么图片还应该继续加载吗？当然不应该。可是Glide并没有办法知道Activity的生命周期，于是Glide就使用了添加隐藏Fragment的这种小技巧，因为Fragment的生命周期和Activity是同步的，如果Activity被销毁了，Fragment是可以监听到的，这样Glide就可以捕获这个事件并停止图片加载了。

那么
#### 如何绑定生命周期
继续接着上面的代码看supportFragmentGet
```java
 RequestManager supportFragmentGet(Context context, FragmentManager fm) {
        SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm);
        RequestManager requestManager = current.getRequestManager();
        if (requestManager == null) {
            requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
            current.setRequestManager(requestManager);
        }
        return requestManager;
    }
```
在fragment 中创建RequestManager，并且传入getLifecycle()，通过Lifecycle实现生命周期的绑定

### load()方法
`Glide.with`获得RequestManager之后，执行load方法设置图片源。图片源可以是：图片数据字节数组、File文件，网络图片地址等。以`load(String)`为例：

```java
//RequestManager
public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
}
public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
}
public <ResourceType> RequestBuilder<ResourceType> as(
      @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
}
```

load方法默认设置目标资源为 **Drawable**，获得一个RequestBuilder。紧接着调用`RequestBuilder.load`方法记录加载的model(图片源)。

```java
//RequestBuilder
public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
}
@NonNull
private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    this.model = model;
    isModelSet = true;
    return this;
}
```

RequestBuilder是请求构建者，用户可以使用它设置如：单独的缓存策略、加载成功前占位图、加载失败后显示图片等等加载图片的各种配置。当RequestBuilder 构建完成之后，接下来就等待执行这个请求。

### into()方法

`Glide.with`获得RequestManager之后，执行load方法设置图片源。图片源可以是：图片数据字节数组、File文件，网络图片地址等。以`load(String)`为例：

```java
//RequestManager
public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
}
public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
}
public <ResourceType> RequestBuilder<ResourceType> as(
      @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
}
```

load方法默认设置目标资源为 **Drawable**，获得一个RequestBuilder。紧接着调用`RequestBuilder.load`方法记录加载的model(图片源)。

```java
//RequestBuilder
public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
}
@NonNull
private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    this.model = model;
    isModelSet = true;
    return this;
}
```

RequestBuilder是请求构建者，用户可以使用它设置如：单独的缓存策略、加载成功前占位图、加载失败后显示图片等等加载图片的各种配置。当RequestBuilder 构建完成之后，接下来就等待执行这个请求。

#### RequestBuilder.into

使用Glide最简单的方式加载图片最后一个阶段就是执行into方法。从into方法为入口开始执行图片加载，逻辑也开始复杂起来。

```java
//RequestBuilder
@NonNull
  public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    Util.assertMainThread();
    Preconditions.checkNotNull(view);

    BaseRequestOptions<?> requestOptions = this;
    if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      // Clone in this method so that if we use this RequestBuilder to load into a View and then
      // into a different target, we don't retain the transformation applied based on the previous
      // View's scale type.
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          requestOptions = requestOptions.clone().optionalFitCenter();
          break;
        case FIT_XY:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case CENTER:
        case MATRIX:
        default:
          // Do nothing.
      }
    }

    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
  }
```

一般来说，我们在执行into时传入一个ImageView用于显示。在这个into方法中，先确定本次加载的BaseRequestOptions，然后执行重载的另一个into方法。其中BaseRequestOptions就是上面我们提到的RequestBuilder可以设置图片加载的各种配置，这些配置选项就被封装在BaseRequestOptions中(RequestBuilder extends BaseRequestOptions)。而重载的into方法实现为:

```java
//GenericRequestBuilder

 public <Y extends Target<TranscodeType>> Y into(Y target) {
        Util.assertMainThread();
        if (target == null) {
            throw new IllegalArgumentException("You must pass in a non null Target");
        }
        if (!isModelSet) {
            throw new IllegalArgumentException("You must first set a model (try #load())");
        }

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
```

这个into方法中首先调用 buildRequest 构建了一个请求Request，然后把Request交给RequestManager跟踪(生命周期)并启动请求。
跟进requestTracker.runRequest()；

```java
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
先判断Glide当前是不是处理暂停状态，如果不是暂停状态就调用Request的begin()方法来执行Request，否则的话就先将Request添加到待执行队列里面，等暂停状态解除了之后再执行。
跟进request.begin();
```java
    /**
     * {@inheritDoc}
     */
    @Override
    public void begin() {
        startTime = LogTime.getLogTime();
        if (model == null) {
            onException(null);
            return;
        }

        status = Status.WAITING_FOR_SIZE;
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
            onSizeReady(overrideWidth, overrideHeight);
        } else {
            target.getSize(this);
        }

        if (!isComplete() && !isFailed() && canNotifyStatusChanged()) {
            target.onLoadStarted(getPlaceholderDrawable());
        }
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logV("finished run method in " + LogTime.getElapsedMillis(startTime));
        }
    }
```
跟进onException(null);
```java
    @Override
    public void onException(Exception e) {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
            Log.d(TAG, "load failed", e);
        }

        status = Status.FAILED;
        //TODO: what if this is a thumbnail request?
        if (requestListener == null || !requestListener.onException(e, model, target, isFirstReadyResource())) {
            setErrorPlaceholder(e);
        }
    }
```
再跟setErrorPlaceholder(e);
```java
  @Override
    public void onException(Exception e) {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
            Log.d(TAG, "load failed", e);
        }

        status = Status.FAILED;
        //TODO: what if this is a thumbnail request?
        if (requestListener == null || !requestListener.onException(e, model, target, isFirstReadyResource())) {
            setErrorPlaceholder(e);
        }
    }

     private void setErrorPlaceholder(Exception e) {
        if (!canNotifyStatusChanged()) {
            return;
        }

        Drawable error = model == null ? getFallbackDrawable() : null;
        if (error == null) {
          error = getErrorDrawable();
        }
        if (error == null) {
            error = getPlaceholderDrawable();
        }
        target.onLoadFailed(e, error);
    }
    ```
其实就是将这张error占位图显示到ImageView上而已，因为现在出现了异常，没办法展示正常的图片了。而如果你仔细看下刚才begin()方法的第15行，你会发现它又调用了一个target.onLoadStarted()方法，并传入了一个loading占位图，在也就说，在图片请求开始之前，会先使用这张占位图代替最终的图片显示。

回到begin()，主要是判断当前Request的各种状态并且设置占位图，而启动加载是在第四步开始的。当用户设置了图片的宽高时执行`onSizeReady`启动加载，而如果未设置则会执行`target.getSize(this)`计算目标View(需要显示的ImageView)的大小，在计算完之后，它也会调用onSizeReady()方法。也就是说，不管是哪种情况，最终都会调用到onSizeReady()方法。

```java
public void onSizeReady(int width, int height) {
    synchronized (requestLock) {
      status = Status.RUNNING;
      loadStatus =
          engine.load(
              glideContext,
              model,
              requestOptions.getSignature(),
              this.width,
              this.height,
              requestOptions.getResourceClass(),
              transcodeClass,
              priority,
              requestOptions.getDiskCacheStrategy(),
              requestOptions.getTransformations(),
              requestOptions.isTransformationRequired(),
              requestOptions.isScaleOnlyOrNoTransform(),
              requestOptions.getOptions(),
              requestOptions.isMemoryCacheable(),
              requestOptions.getUseUnlimitedSourceGeneratorsPool(),
              requestOptions.getUseAnimationPool(),
              requestOptions.getOnlyRetrieveFromCache(),
              this,
              callbackExecutor);
      //.....
    }
}

```

在onSizeReady将获得一系列的值一起传入到了Engine的load()方法当中开始加载图片。

**Engine**类顾名思义:引擎，是Glide加载的发动机，跟进Engine的load()

```java
 public <R> LoadStatus load(...) {
   ...
    //1、根据参数（模型、宽、高等）构建加载的标识key
    EngineKey key =
        keyFactory.buildKey(
            model,
            signature,
            width,
            height,
            transformations,
            resourceClass,
            transcodeClass,
            options);

    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return null;
    }

    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return null;
    }

    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      current.addCallback(cb, callbackExecutor);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Added to existing load", startTime, key);
      }
      return new LoadStatus(cb, current);
    }
//2、创建新的job并执行加载
    EngineJob<R> engineJob =
        engineJobFactory.build(
            key,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache);
 // DecodeJob就是一个Runnable,EngineJob包含的线程池启动线程并能够接收线程中执行过程的回调
    DecodeJob<R> decodeJob =
        decodeJobFactory.build(
            glideContext,
            model,
            key,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            onlyRetrieveFromCache,
            options,
            engineJob);

    jobs.put(key, engineJob);

    engineJob.addCallback(cb, callbackExecutor);
    // 启动线程
    engineJob.start(decodeJob);
    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
}
```

如果引擎中没有同样的正在加载的请求，此处会创建一个实现了Runable接口的DecodeJob并执行。
跟进engineJob.start(decodeJob);

```java
//EngineJob 
public synchronized void start(DecodeJob<R> decodeJob) {
    this.decodeJob = decodeJob;
    //线程池
    GlideExecutor executor =
        decodeJob.willDecodeFromCache() ? diskCacheExecutor : getActiveSourceExecutor();
    executor.execute(decodeJob);
}
```

使用线程池执行任务DecodeJob。

> 如果需要获取一张图片，那么先从内存缓存中获得，内存缓存不需要进行额外的操作，只要匹配就能直接获取。但是内存缓存不存在，这时候就需要先检查磁盘缓存最后再决定是否需要向图片源(如网络)进行加载。而无论是进行图片磁盘缓存的获取还是向网络请求都是耗时操作，这时候就需要开启线程来执行加载。因此内存缓存一旦不存在，就会使用线程池开启DecodeJob异步任务。而DecodeJob中就会去磁盘缓存查找或者网络中下载图片。

DecodeJob是实现了Runable接口的一个任务，当使用线程去执行这个任务，自然就会执行run方法

```java
//DecodeJob

public void run() {
    //...
      runWrapped();
    //...
  }
```

run方法中除了取消与加载失败的判断，核心的加载逻辑都放在了runWrapped中。在介绍这个方法之前，先简单的了解下Glide中磁盘缓存。 Glide可以缓存两种不同的数据在磁盘文件中：

- 1 资源类型（Resource） - 曾被解码、转换并写入的磁盘缓存。如对原图片进行了缩放，存放缩放后的图片。
- 2 数据来源 (Data) - 原图片缓存

而runWrapped方法的实现为：

```java
//DecodeJob

private void runWrapped() {
    switch (runReason) {
      case INITIALIZE: 
        // 1. 获取任务的场景
        stage = getNextStage(Stage.INITIALIZE);
        // 2. 获取这个场景的执行者
        currentGenerator = getNextGenerator();
        // 3. 执行者执行任务
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE:
        // 当图片数据无法从缓存加载时，会切换执行原因并记录，然后重新提交本任务给线程池
        // （使用另一个线程池执行图片源加载，如：网络加载任务）
        runGenerators();
        break;
      case DECODE_DATA:
        // 切换为解码图片,获得的图片数据，比如网络下载了 a.png 得到其byte数组，
        // byte数组中记录的是png编码后的数据，BitmapFactory.decodeXX 会解码生成Bitmap对象
        // 同时解码过程中可能还需要进行缩放、变换等操作    
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
  }

private Stage getNextStage(Stage current) {
    switch (current) {
       // 1 如果配置的缓存策略允许从 资源缓存 中读数据, 则返回 Stage.RESOURCE_CACHE
      case INITIALIZE:
        return diskCacheStrategy.decodeCachedResource()
            ? Stage.RESOURCE_CACHE
            : getNextStage(Stage.RESOURCE_CACHE);
      case RESOURCE_CACHE:
        // 2 如果配置的缓存策略允许从 源数据缓存 中读数据, 则返回 Stage.DATA_CACHE
        return diskCacheStrategy.decodeCachedData()
            ? Stage.DATA_CACHE
            : getNextStage(Stage.DATA_CACHE);
      case DATA_CACHE:
        // 3 如果只允许从缓存中获取数据, 则直接 FINISH, 否则返回 Stage.SOURCE,表示加载一个新的资源
        return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
      case SOURCE:
      case FINISHED:
        return Stage.FINISHED;
      default:
        throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
  }

  private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
      // 资源磁盘缓存的执行者
      case RESOURCE_CACHE:
        return new ResourceCacheGenerator(decodeHelper, this);
     // 源数据磁盘缓存的执行者
      case DATA_CACHE:
        return new DataCacheGenerator(decodeHelper, this);
      case SOURCE:
     // 无缓存, 获取数据的源的执行者
        return new SourceGenerator(decodeHelper, this);
      case FINISHED:
        return null;
      default:
        throw new IllegalStateException("Unrecognized stage: " + stage);
    }
  }

  private void runGenerators() {
    //......
    // 调用 DataFetcherGenerator.startNext() 执行请求操作
    boolean isStarted = false;
    while (!isCancelled
        && currentGenerator != null
        && !(isStarted = currentGenerator.startNext())) {
      //如果执行者未执行，获取下一个场景的执行者
      stage = getNextStage(stage);
      currentGenerator = getNextGenerator();

      if (stage == Stage.SOURCE) {
        reschedule();
        return;
      }
    }
    // We've run out of stages and generators, give up.
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
      notifyFailed();
    }
}


```

DecodeJob 任务执行时, 它根据不同的场景, 获取不同的场景执行器, 然后调用了它们的 startNext 方法加载请求任务的数据。

| 场景                 | 场景描述                     | 场景执行者             |
| -------------------- | ---------------------------- | ---------------------- |
| Stage.RESOURCE_CACHE | 从磁盘中获取转换后的图片缓存 | ResourceCacheGenerator |
| Stage.DATA_CACHE     | 从磁盘中获取原图片缓存数据   | DataCacheGenerator     |
| Stage.SOURCE         | 图片源请求数据               | SourceGenerator        |



总的来说，Glide在加载图片的时候，会首先从内存缓存获取，如果内存缓存不存在，则会在磁盘缓存中查找，否则从源地址进行加载。

## Glide缓存
简单来说分为 活动缓存（弱引用activeResources），内存缓存（LruResourceCache），资源缓存，数据缓存

### 活动缓存和内存缓存
默认情况下，Glide自动就是开启内存缓存的。也就是说，当我们使用Glide加载了一张图片之后，这张图片就会被缓存到内存当中，只要在它还没从内存中被清除之前，下次使用Glide再加载这张图片都会直接从内存当中读取，而不用重新从网络或硬盘上读取了，这样无疑就可以大幅度提升图片的加载效率。比方说你在一个RecyclerView当中反复上下滑动，RecyclerView中只要是Glide加载过的图片都可以直接从内存当中迅速读取并展示出来，从而大大提升了用户体验。不过可以通过这种方式禁掉默认缓存：
```
Glide.with(this)
     .load(url)
     .skipMemoryCache(true)
     .into(imageView);
//skipMemoryCache传true就可以啦
```
那么看一看原理：
load()方法中，我们当时分析到了在loadGeneric()方法中会调用Glide.buildStreamModelLoader()方法来获取一个ModelLoader对象
```java
 public static <T, Y> ModelLoader<T, Y> buildModelLoader(Class<T> modelClass, Class<Y> resourceClass,
            Context context) {
         if (modelClass == null) {
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "Unable to load null model, setting placeholder only");
            }
            return null;
        }
        return Glide.get(context).getLoaderFactory().buildModelLoader(modelClass, resourceClass);
    }

    public static Glide get(Context context) {
        if (glide == null) {
            synchronized (Glide.class) {
                if (glide == null) {
                    Context applicationContext = context.getApplicationContext();
                    List<GlideModule> modules = new ManifestParser(applicationContext).parse();
                    GlideBuilder builder = new GlideBuilder(applicationContext);
                    for (GlideModule module : modules) {
                        module.applyOptions(applicationContext, builder);
                    }
                    glide = builder.createGlide();
                    for (GlideModule module : modules) {
                        module.registerComponents(applicationContext, glide);
                    }
                }
            }
        }
        return glide;
    }

```
跟createGlide（）方法
```java
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

```
而 memoryCache = new LruResourceCache(calculator.getMemoryCacheSize()); 就是Glide实现内存缓存所使用的LruCache对象了，有对象还不够，关键在于Engine类load()方法，在load()方法中Glide的图片加载过程中会调用两个方法来获取内存缓存，loadFromCache()和loadFromActiveResources()。这两个方法中一个使用的就是LruCache算法，另一个使用的就是弱引用。我们来看一下它们的源码：
```java
public class Engine implements EngineJobListener,
        MemoryCache.ResourceRemovedListener,
        EngineResource.ResourceListener {

    private final MemoryCache cache;
    private final Map<Key, WeakReference<EngineResource<?>>> activeResources;
    ...

    private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }
        EngineResource<?> cached = getEngineResourceFromCache(key);
        if (cached != null) {
            cached.acquire();
            activeResources.put(key, new ResourceWeakReference(key, cached, getReferenceQueue()));
        }
        return cached;
    }

    private EngineResource<?> getEngineResourceFromCache(Key key) {
        Resource<?> cached = cache.remove(key);
        final EngineResource result;
        if (cached == null) {
            result = null;
        } else if (cached instanceof EngineResource) {
            result = (EngineResource) cached;
        } else {
            result = new EngineResource(cached, true /*isCacheable*/);
        }
        return result;
    }

    private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }
        EngineResource<?> active = null;
        WeakReference<EngineResource<?>> activeRef = activeResources.get(key);
        if (activeRef != null) {
            active = activeRef.get();
            if (active != null) {
                active.acquire();
            } else {
                activeResources.remove(key);
            }
        }
        return active;
    }

    ...
}

```
在loadFromCache()方法的一开始，首先就判断了isMemoryCacheable是不是false，如果是false的话就直接返回null，意思是刚刚的skipMemoryCache()方法，如果在这个方法中传入true，那么这里的isMemoryCacheable就会是false，表示内存缓存已被禁用。

我们继续住下看，接着调用了getEngineResourceFromCache()方法来获取缓存。在这个方法中，会使用缓存Key来从cache当中取值，而这里的cache对象就是在构建Glide对象时创建的LruResourceCache，那么说明这里其实使用的就是LruCache算法了。

但是，当我们从LruResourceCache中获取到缓存图片之后会将它从缓存中移除，然后将这个缓存图片存储到activeResources当中。activeResources就是一个弱引用的HashMap，用来缓存正在使用中的图片，我们可以看到，loadFromActiveResources()方法就是从activeResources这个HashMap当中取值的。使用activeResources来缓存正在使用中的图片，可以保护这些图片不会被LruCache算法回收掉。

概括一下来说，就是如果能从内存缓存当中读取到要加载的图片，那么就直接进行回调，如果读取不到的话，才会开启线程执行后面的图片加载逻辑。

### 资源缓存，数据缓存
用法上，它也是可禁的
```
Glide.with(this)
     .load(url)
     .diskCacheStrategy(DiskCacheStrategy.NONE)
     .into(imageView);
```
调用diskCacheStrategy()方法并传入DiskCacheStrategy.NONE，就可以禁用掉Glide的硬盘缓存功能了。

这个diskCacheStrategy()方法基本上就是Glide硬盘缓存功能的一切，它可以接收四种参数：

DiskCacheStrategy.NONE： 表示不缓存任何内容。
DiskCacheStrategy.SOURCE： 表示只缓存原始图片。
DiskCacheStrategy.RESULT： 表示只缓存转换过后的图片（默认选项）。
DiskCacheStrategy.ALL ： 表示既缓存原始图片，也缓存转换过后的图片。
上面四种参数的解释本身并没有什么难理解的地方，当我们使用Glide去加载一张图片的时候，Glide默认并不会将原始图片展示出来，而是会对图片进行压缩和转换（我们会在后面学习这方面的内容）。总之就是经过种种一系列操作之后得到的图片，就叫转换过后的图片。而Glide默认情况下在硬盘缓存的就是转换过后的图片，我们通过调用diskCacheStrategy()方法则可以改变这一默认行为。

接下来还是通过阅读源码来分析一下，Glide的硬盘缓存功能是如何实现的。
首先，和内存缓存类似，硬盘缓存的实现也是使用的LruCache算法，而且Google还提供了一个现成的工具类DiskLruCache
，Glide开启线程来加载图片后会执行EngineRunnable的run()方法，run()方法中又会调用一个decode()方法，那么我们重新再来看一下这个decode()方法的源码：
```java
private Resource<?> decode() throws Exception {
    if (isDecodingFromCache()) {
        return decodeFromCache();
    } else {
        return decodeFromSource();
    }
}

```
可以看到，这里会分为两种情况，一种是调用decodeFromCache()方法从硬盘缓存当中读取图片，一种是调用decodeFromSource()来读取原始图片。默认情况下Glide会优先从缓存当中读取，只有缓存中不存在要读取的图片时，才会去读取原始图片。那么我们现在来看一下decodeFromCache()方法的源码，如下所示：
```java
private Resource<?> decodeFromCache() throws Exception {
    Resource<?> result = null;
    try {
        result = decodeJob.decodeResultFromCache();
    } catch (Exception e) {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
            Log.d(TAG, "Exception decoding result from cache: " + e);
        }
    }
    if (result == null) {
        result = decodeJob.decodeSourceFromCache();
    }
    return result;
}
```
可以看到，这里会先去调用DecodeJob的decodeResultFromCache()方法来获取缓存，如果获取不到，会再调用decodeSourceFromCache()方法获取缓存，这两个方法的区别其实就是DiskCacheStrategy.RESULT(压缩后的)和DiskCacheStrategy.SOURCE(原始的)
看一下这两个方法的源码
```java
public Resource<Z> decodeResultFromCache() throws Exception {
    if (!diskCacheStrategy.cacheResult()) {
        return null;
    }
    long startTime = LogTime.getLogTime();
    Resource<T> transformed = loadFromCache(resultKey);
    startTime = LogTime.getLogTime();
    Resource<Z> result = transcode(transformed);
    return result;
}

public Resource<Z> decodeSourceFromCache() throws Exception {
    if (!diskCacheStrategy.cacheSource()) {
        return null;
    }
    long startTime = LogTime.getLogTime();

    Resource<T> decoded = loadFromCache(resultKey.getOriginalKey());
    return transformEncodeAndTranscode(decoded);
}

```
可以看到，它们都是调用了loadFromCache()方法从缓存当中读取数据，如果是decodeResultFromCache()方法就直接将数据解码并返回，如果是decodeSourceFromCache()方法，还要调用一下transformEncodeAndTranscode()方法先将数据转换一下再解码并返回。
看一下loadFromCache()方法的源码
```java
private Resource<T> loadFromCache(Key key) throws IOException {
    File cacheFile = diskCacheProvider.getDiskCache().get(key);
    if (cacheFile == null) {
        return null;
    }
    Resource<T> result = null;
    try {
        result = loadProvider.getCacheDecoder().decode(cacheFile, width, height);
    } finally {
        if (result == null) {
            diskCacheProvider.getDiskCache().delete(key);
        }
    }
    return result;
}

```
这个方法的逻辑非常简单，调用getDiskCache()方法获取到的就是Glide自己编写的DiskLruCache工具类的实例，然后调用它的get()方法并把缓存Key传入，就能得到硬盘缓存的文件了。如果文件为空就返回null，如果文件不为空则将它解码成Resource对象后返回即可。

### 看源码之前不清楚的两个问题：

- 1 Glide的方法能在子线程中使用吗？
> Glide的with()方法和load()方法是可以的，其中with()传入的context，若在非主线程，那么不管你是传入的Activity还是Fragment，都会被强制当成Application来处理，可能造成内存泄漏；而into()无法在主线程中使用，会抛出异常“You must call this method on the main thread”

- 2 如何处理一个大图的缓存？
```java
BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;//只读取图片，不加载到内存中
        BitmapFactory.decodeFile(file, options);
        options.inSampleSize=computeSampleSize(options, -1, 512*512);//返回合适的inSampleSize值,inSampleSize是缩放参数，必须被二整除
        options.inJustDecodeBounds = false;//加载到内存中
        bitmap = BitmapFactory.decodeFile(file, options);
```







