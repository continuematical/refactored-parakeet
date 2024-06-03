# Glide

Glide是一个安全高效的图片加载库，注重于平滑的滚动，提供了易用的API，可扩展的图片解码通道，支持**拉取，解码和展示视频快照，图片和GIF动画**。默认情况下，Glide使用的是一个**HttpUriConnection**的栈，同时提供了与Google Valley和Square okHttp快速集成的工具库。

## 使用

```java
Glide.with(this).load(R.drawable.image).into(mImageView);
```

### with()

返回一个`RequestManager` 对象；

```java
//with()
public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
    //	获得一个RequestManagerRetriever
    //	获得一个RequestManager
  }
```

```java
//getRetriever()
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    Preconditions.checkNotNull(
        context,
        "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
            + "returns null (which usually occurs when getActivity() is called before the Fragment"
            + "is attached or after the Fragment is destroyed).");
    return Glide.get(context).getRequestManagerRetriever();
    //			获得一个Glide对象	RequestManagerRetriever的构造方法
```

```java
//get()	获得一个Glide对象
public static Glide get(@NonNull Context context) {
    if (glide == null) {
      GeneratedAppGlideModule annotationGeneratedModule =
          //防止内存泄漏
          getAnnotationGeneratedGlideModules(context.getApplicationContext());
      synchronized (Glide.class) {
        if (glide == null) {
            // 对glide整体地进行初始化
            // 其中就包含了对RequestManagerRetriever的初始化流程
          checkAndInitializeGlide(context, annotationGeneratedModule);
        }
      }
    }

    return glide;
  }
```

```java
//RequestManagerRetriever
public RequestManagerRetriever(
      @Nullable RequestManagerFactory factory, GlideExperiments experiments) {
    this.factory = factory != null ? factory : DEFAULT_FACTORY;
    //通信工具为Handle，最后回到主线程
    handler = new Handler(Looper.getMainLooper(), this /* Callback */);
    lifecycleRequestManagerRetriever = new LifecycleRequestManagerRetriever(this.factory);
    frameWaiter = buildFrameWaiter(experiments);
  }

//RequestManagerFactory
private static final RequestManagerFactory DEFAULT_FACTORY =
      new RequestManagerFactory() {
        @NonNull
        @Override
        public RequestManager build(
            @NonNull Glide glide,
            @NonNull Lifecycle lifecycle,
            @NonNull RequestManagerTreeNode requestManagerTreeNode,
            @NonNull Context context) {
          return new RequestManager(glide, lifecycle, requestManagerTreeNode, context);
        }
      };
```

```java
//getRetriever(activity).get(activity)
public RequestManager get(@NonNull FragmentActivity activity) {
    //如果在后台进程，则进入
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    }
    assertNotDestroyed(activity);
    frameWaiter.registerSelf(activity);
    boolean isActivityVisible = isActivityVisible(activity);
    Glide glide = Glide.get(activity.getApplicationContext());
    //最后都是为了创建出一个RequestManager
    return lifecycleRequestManagerRetriever.getOrCreate(
        activity,
        glide,
        activity.getLifecycle(),
        activity.getSupportFragmentManager(),
        isActivityVisible);
  }
```

Glide不能在子线程中`with()` 因为子线程不会添加生命周期管理机制，主线程才会生成一个空白的Fragment监听生命周期变化；

`get()` 的重载方法

```java
public RequestManager get(@NonNull Context context) {
    if (context == null) {
      throw new IllegalArgumentException("You cannot start a load on a null Context");
        //不是Application且在主线程中时
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
      if (context instanceof FragmentActivity) {
        return get((FragmentActivity) context);
      } else if (context instanceof Activity) {
        return get((Activity) context);
      } else if (context instanceof ContextWrapper
          && ((ContextWrapper) context).getBaseContext().getApplicationContext() != null) {
        return get(((ContextWrapper) context).getBaseContext());
      }
    }
	//是Application且不在主线程中
    return getApplicationManager(context);
  }
```

```java
//getApplicationManager(context)
private RequestManager getApplicationManager(@NonNull Context context) {
    //使用DCL的方式来创建单例
    //双重锁定检查，在多线程环境下使用
    if (applicationManager == null) {
      synchronized (this) {
        if (applicationManager == null) {
          Glide glide = Glide.get(context.getApplicationContext());
            //工厂方式创建对象
          applicationManager =
              factory.build(
                  glide,
              //ApplicationLifecycle某些情况下会接受不到生命周期的事件，这里是做的强制性的操作是为了生命周期变化时能够正常相应，将生命周期的监听与Application强制绑定用于接收。
                  new ApplicationLifecycle(),
                  new EmptyRequestManagerTreeNode(),
                  context.getApplicationContext());
        }
      }
    }
    return applicationManager;
  }
```

### load()

支持网络资源，resource资源，File资源，Uri资源；

返回一个`requestBuilder` 对象，加载过程放在`into()` 里；

```java
//将获取的数据转化为Drawble类然后在ImageView上显示
public RequestBuilder<Drawable> load(@Nullable Uri uri) {
    return asDrawable().load(uri);
  }
```

```java
//asDrawable()
public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
  }
```

```java
//as
//完成RequestBuilder的创建
public <ResourceType> RequestBuilder<ResourceType> as(
      @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
  }
```

```java
//RequestBuilder
protected RequestBuilder(
      @NonNull Glide glide,
      RequestManager requestManager,
      Class<TranscodeType> transcodeClass,
      Context context) {
    this.glide = glide;
    this.requestManager = requestManager;
    this.transcodeClass = transcodeClass;
    this.context = context;
    this.transitionOptions = requestManager.getDefaultTransitionOptions(transcodeClass);
    this.glideContext = glide.getGlideContext();
    //从某种意义上讲就是对生命周期的监听
    initRequestListeners(requestManager.getDefaultRequestListeners());
    //它的主要任务就是一些策略开关
    // 各种选项的开启装置，比如错误提示、优先级、磁盘缓存策略、固定宽高等等
    apply(requestManager.getDefaultRequestOptions());
  }
```

```java
//initRequestListeners
private void initRequestListeners(List<RequestListener<Object>> requestListeners) {
    for (RequestListener<Object> listener : requestListeners) {
      addListener((RequestListener<TranscodeType>) listener);
    }
  }
```

```java
//asDrawable().load(uri)
public RequestBuilder<TranscodeType> load(@Nullable Uri uri) {
    return maybeApplyOptionsResourceUri(uri, loadGeneric(uri));
  }
```

```java
//loadGeneric
private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    if (isAutoCloneEnabled()) {
      return clone().loadGeneric(model);
    }
    this.model = model;
    isModelSet = true;
    return selfOrThrowIfLocked();
  }
```

将`URI`的值放到了`model`这个变量中，那整个`load()`函数作用其实最后只是创建了一个`RequestBuilder`的事例.

### into()

```java
//into()
public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    Util.assertMainThread();
    Preconditions.checkNotNull(view);

    //检测ScaleType形式
    BaseRequestOptions<?> requestOptions = this;
    if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      //通过ScaleType对图片进行设置
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case FIT_CENTER:
        case FIT_START:
        //没有设置，默认情况下
        case FIT_END:
          //原型设计模式
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
	//塞入图片
    return into(//1--
        //对View进行了赋值
        glideContext.buildImageViewTarget(view, transcodeClass),
        /* targetListener= */ null,
        requestOptions,
        //使用线程池获取和处理图片
        Executors.mainThreadExecutor());
  }
```

```java
//1--into()
private <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,
      Executor callbackExecutor) {
    Preconditions.checkNotNull(target);
    if (!isModelSet) {
      throw new IllegalArgumentException("You must call #load() before calling #into()");
    }

    // 正常情况构建SingleRequest的请求
    // 因为thumbnail一般需要额外的需求
    //接口
    Request request = buildRequest(target, targetListener, options, callbackExecutor);//2--
    Request previous = target.getRequest();
    //当前请求和最新的一样
    if (request.isEquivalentTo(previous)
        && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
      // 如果请求完成，重新启动会保证结果送达并触动目标
      // 如果请求失败，会给出机会去再次完成
      // 如果请求正在运行，不会打断
      if (!Preconditions.checkNotNull(previous).isRunning()) {
        previous.begin();
      }
      return target;
    }
    
    //保证是最新的请求
    requestManager.clear(target);
    target.setRequest(request);//置换最新的请求
    requestManager.track(target, request);//6--

    return target;
  }
```

```java
//2--glideContext.buildImageViewTarget(view, transcodeClass)
//构建一个存放的目标
public <X> ViewTarget<ImageView, X> buildImageViewTarget(
      @NonNull ImageView imageView, @NonNull Class<X> transcodeClass) {
    return imageViewTargetFactory.buildTarget(imageView, transcodeClass);//3--
  }
```

```java
//3--buildTarget()
public class ImageViewTargetFactory {
  @NonNull
  @SuppressWarnings("unchecked")
  public <Z> ViewTarget<ImageView, Z> buildTarget(
      @NonNull ImageView view, @NonNull Class<Z> clazz) {
      // 根据不同的数据类型选择存储是以Drawable还是Bitmap构建
    if (Bitmap.class.equals(clazz)) {
      return (ViewTarget<ImageView, Z>) new BitmapImageViewTarget(view);//4--
    } else if (Drawable.class.isAssignableFrom(clazz)) {
      return (ViewTarget<ImageView, Z>) new DrawableImageViewTarget(view);//5--
    } else {
      throw new IllegalArgumentException(
          "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
    }
  }
}
```

```java
//4,5--两个方法都会调用
public ViewTarget(@NonNull T view) {
    this.view = Preconditions.checkNotNull(view);
    sizeDeterminer = new SizeDeterminer(view);
  }
//对View进行了一个赋值操作
SizeDeterminer(@NonNull View view) {
      this.view = view;
    }
```

```java
//6--requestManager.track(target, request)
//以同步的方式来完成数据请求
synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
    targetTracker.track(target);//对当前的生命周期完成一个追踪
    requestTracker.runRequest(request);//7--执行操作开始
  }
```

```java
//7--
public void runRequest(@NonNull Request request) {
    //集合Set
    requests.add(request);//正在运行中的队列
    // 会对当前的所有请求做一个判断处理
    // 会根据当前的状态确定是否要进行数据加载的操作
    // 一般来说对应的就是生命周期
    if (!isPaused) {//未暂停
      request.begin();//运行8--
    } else {//暂停
      request.clear();//清除
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Paused, delaying request");
      }
      //List
      pendingRequests.add(request);//等待中的队列
    }
  }
```

```java
//8--SingleRequest.begin
public void begin() {
    //多线程的安全性
    synchronized (requestLock) {
      assertNotCallingCallbacks();
      stateVerifier.throwIfRecycled();
      startTime = LogTime.getLogTime();
      if (model == null) {
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
          width = overrideWidth;
          height = overrideHeight;
        }
        int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
        onLoadFailed(new GlideException("Received null model"), logLevel);
        return;
      }

        //正常运行，抛出异常
      if (status == Status.RUNNING) {
        throw new IllegalArgumentException("Cannot restart a running request");
      }
        //从缓存中直接拿数据
      if (status == Status.COMPLETE) {
        onResourceReady(
            resource, DataSource.MEMORY_CACHE, /* isLoadedFromAlternateCacheKey= */ false);
        return;
      }
      experimentalNotifyRequestStarted(model);

      cookie = GlideTrace.beginSectionAsync(TAG);
        //提供解决图片大小的方案，给出固定长宽/未设置大小
      status = Status.WAITING_FOR_SIZE;
        //系统自动测量的长宽
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        onSizeReady(overrideWidth, overrideHeight);//设置图片大小9--
      } else {
        target.getSize(this);
      }
		//使用一个占位符顶替
      if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
          && canNotifyStatusChanged()) {
        target.onLoadStarted(getPlaceholderDrawable());
      }
      if (IS_VERBOSE_LOGGABLE) {
        logV("finished run method in " + LogTime.getElapsedMillis(startTime));
      }
    }
  }
```

```java
//9--onSizeReady(overrideWidth, overrideHeight)
public void onSizeReady(int width, int height) {
    stateVerifier.throwIfRecycled();
    synchronized (requestLock) {
      if (IS_VERBOSE_LOGGABLE) {
        logV("Got onSizeReady in " + LogTime.getElapsedMillis(startTime));
      }
      if (status != Status.WAITING_FOR_SIZE) {
        return;
      }
      status = Status.RUNNING;
	//对长宽进行预估
      float sizeMultiplier = requestOptions.getSizeMultiplier();
      this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
      this.height = maybeApplySizeMultiplier(height, sizeMultiplier);

      if (IS_VERBOSE_LOGGABLE) {
        logV("finished setup for calling load in " + LogTime.getElapsedMillis(startTime));
      }
      loadStatus =
          engine.load(//10--
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
      if (status != Status.RUNNING) {
        loadStatus = null;
      }
      if (IS_VERBOSE_LOGGABLE) {
        logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
      }
    }
  }
```

```java
//10--load
public <R> LoadStatus load(
      GlideContext glideContext,
      Object model,
      Key signature,
      int width,
      int height,
      Class<?> resourceClass,
      Class<R> transcodeClass,
      Priority priority,
      DiskCacheStrategy diskCacheStrategy,
      Map<Class<?>, Transformation<?>> transformations,
      boolean isTransformationRequired,
      boolean isScaleOnlyOrNoTransform,
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean useAnimationPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb,
      Executor callbackExecutor) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
//工厂模式，缓存机制的唯一标识
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

    EngineResource<?> memoryResource;
    synchronized (this) {
     //缓存机制
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);//1--
	//缓存机制为空
      if (memoryResource == null) {
        return waitForExistingOrStartNewJob(//2--
            glideContext,
            model,
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
            options,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache,
            cb,
            callbackExecutor,
            key,
            startTime);
      }
    }
    //缓存机制不为空，直接使用
    cb.onResourceReady(
        memoryResource, DataSource.MEMORY_CACHE, /* isLoadedFromAlternateCacheKey= */ false);
    return null;
  }
```

```java
//2--loadFromMemory()
//缓存机制
private EngineResource<?> loadFromMemory(
      EngineKey key, boolean isMemoryCacheable, long startTime) {
    if (!isMemoryCacheable) {
      return null;
    }
//活动缓存--一级缓存
    EngineResource<?> active = loadFromActiveResources(key);
    if (active != null) {
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return active;
    }
//内存缓存--二级缓存
//服务于活动缓存
    EngineResource<?> cached = loadFromCache(key);
    if (cached != null) {
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return cached;
    }
    return null;
  }
```

$$
运行时缓存
\begin{cases}
一级缓存\\
二级缓存\\
\end{cases}
$$

```java
//2--waitForExistingOrStartNewJob
private <R> LoadStatus waitForExistingOrStartNewJob(
      GlideContext glideContext,
      Object model,
      Key signature,
      int width,
      int height,
      Class<?> resourceClass,
      Class<R> transcodeClass,
      Priority priority,
      DiskCacheStrategy diskCacheStrategy,
      Map<Class<?>, Transformation<?>> transformations,
      boolean isTransformationRequired,
      boolean isScaleOnlyOrNoTransform,
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean useAnimationPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb,
      Executor callbackExecutor,
      EngineKey key,
      long startTime) {

    //再次检查运行时是否有可用缓存
    //检测是否有磁盘缓存，如果没有，就要网络请求
    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      current.addCallback(cb, callbackExecutor);
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Added to existing load", startTime, key);
      }
      return new LoadStatus(cb, current);
    }
//工作驱动
    EngineJob<R> engineJob =
        engineJobFactory.build(
            key,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache);
//解码
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

    jobs.put(key, engineJob);//对当前前驱工作进行缓存操作

    engineJob.addCallback(cb, callbackExecutor);
    engineJob.start(decodeJob);//开启图片获取操作

    if (VERBOSE_IS_LOGGABLE) {
      logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
  }
```

```java
//DecodeJob实现Runnable接口
public void run() {
    GlideTrace.beginSectionFormat("DecodeJob#run(reason=%s, model=%s)", runReason, model)
    DataFetcher<?> localFetcher = currentFetcher;
    try {
      if (isCancelled) {
        notifyFailed();
        return;
      }
      runWrapped();//3--
    } catch (CallbackException e) {
      throw e;
    } catch (Throwable t) {
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(
            TAG,
            "DecodeJob threw unexpectedly" + ", isCancelled: " + isCancelled + ", stage: " + stage,
            t);
      
      if (stage != Stage.ENCODE) {
        throwables.add(t);
        notifyFailed();
      }
      if (!isCancelled) {
        throw t;
      }
      throw t;
    } finally {
      if (localFetcher != null) {
        localFetcher.cleanup();
      }
      GlideTrace.endSection();
    }
  }
```

```java
//3--runWrapped()
private void runWrapped() {
    switch (runReason) {
      case INITIALIZE:
        // 获取 Stage.INITIALIZE 的下一步操作，也就是选择三种方案
        // 1. 资源缓存：ResourceCacheGenerator
        // 2. 数据缓存：DataCacheGenerator
        // 3. 网络资源：SourceGenerator
        stage = getNextStage(Stage.INITIALIZE);//4--
        currentGenerator = getNextGenerator();
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE:
        runGenerators();
        break;
      case DECODE_DATA:
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
  }
```

```java
//4--getNextStage
private Stage getNextStage(Stage current) {
    switch (current) {
      case INITIALIZE:
        return diskCacheStrategy.decodeCachedResource()
            ? Stage.RESOURCE_CACHE
            : getNextStage(Stage.RESOURCE_CACHE);
      case RESOURCE_CACHE:
        return diskCacheStrategy.decodeCachedData()
            ? Stage.DATA_CACHE
            : getNextStage(Stage.DATA_CACHE);
      case DATA_CACHE:
        return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
      case SOURCE:////没有配置任何缓存策略，使用默认Source
      case FINISHED:
        return Stage.FINISHED;
      default:
        throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
  }
```



### 占位图

*placeHolder()*	正在请求图片时展出的图片；

*error()*	图片请求失败时展出的图片（如果没有设置的话还是展示placeholder的占位符）；

*fallback()*	请求的Url为null时展出的图片（如果没有设置的话还是展示placeholder的占位符）；

```java
RequestOptions options=new RequestOptions()
                .placeholder(R.drawable.ic_launcher_foreground)
                .error(R.drawable.ic_launcher_background)
                .override(100,100);//设置图片大小
        Glide.with(this)
                .load(R.drawable.image)
            	.apply(options)//apply方法可以调用多次，因此RequestOptions可以多次使用
                .into(mImageView);
```

### 选项

#### RequestOption请求选项

可以设置Glide的大部分属性，让应用的不同部分之间共享相同的加载选项。

#### TransitionOptions过渡选项

设置从占位符到新加载的图片、或从缩略图到全尺寸的图像的过滤；

#### 变换

***圆角配置***

```java
.transform(new CircleGrop())
```

`CircleGrop` 圆角

`RoundedCorner` 四个角度统一指定

`Rotate` 旋转

`GranularRoundedCorners` 四个角度单独指定

## 缓存机制加载流程图

### LruCathe算法

LruCathe算法使用LinkedHashMap数据结构的特性，再加上对LinkedHashMap数据操作上锁的特性。

```java
public LruCache(int maxSize) {
        if (maxSize <= 0) {
            throw new IllegalArgumentException("maxSize <= 0");
        }
        this.maxSize = maxSize;
        this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
    //accessOrder：true使用LruCathe算法，即选择访问排序。
    }
```



当内存缓存机制缓存已满时，调用`trimMemory()` 删除加载次数最少的图片；

```java
public void trimMemory(int level) {
    // Engine asserts this anyway when removing resources, fail faster and consistently
    Util.assertMainThread();
    // Request managers need to be trimmed before the caches and pools, in order for the latter to
    // have the most benefit.
    synchronized (managers) {
      for (RequestManager manager : managers) {
        manager.onTrimMemory(level);
      }
    }
    // memory cache needs to be trimmed before bitmap pool to trim re-pooled Bitmaps too. See #687.
    memoryCache.trimMemory(level);
    bitmapPool.trimMemory(level);
    arrayPool.trimMemory(level);
  }
```

`DiskCathe()` 接口

```java
public interface DiskCache {
  interface Factory {
    int DEFAULT_DISK_CACHE_SIZE = 250 * 1024 * 1024;
    String DEFAULT_DISK_CACHE_DIR = "image_manager_disk_cache";
    @Nullable
    DiskCache build();
  }
  interface Writer {
    boolean write(@NonNull File file);
      //写入成功返回true
  }

  @Nullable
  File get(Key key);
  void put(Key key, Writer writer);
    
  @SuppressWarnings("unused")
  void delete(Key key);

  void clear();
}
```

