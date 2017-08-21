# Glide 学习
Glide is a fast and efficient open source media management and image loading framework for Android that wraps media decoding, memory and disk caching, and resource pooling into a simple and easy to use interface.

* [项目地址](https://github.com/bumptech/glide)

# 分析Glide.with().load().into()过程

## Glide的with
```java
  // 该方法有多个重载，返回一个RequestManager
  public static RequestManager with(Context context) {
    return getRetriever(context).get(context);
  }
```

## Glide的getRetriever
```java
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    // Context could be null for other reasons (ie the user passes in null), but in practice it will
    // only occur due to errors with the Fragment lifecycle.
    Preconditions.checkNotNull(
        context,
        "You cannot start a load on a not yet attached View or a  Fragment where getActivity() "
            + "returns null (which usually occurs when getActivity() is called before the Fragment "
            + "is attached or after the Fragment is destroyed).");
    return Glide.get(context).getRequestManagerRetriever();
}
```
先检查context是否为null，接着调用Glide.get(context).getRequestManagerRetriever()

## Glide的get
```java
public static Glide get(Context context) {
    if (glide == null) {
      synchronized (Glide.class) {
        if (glide == null) {
          checkAndInitializeGlide(context);
        }
      }
    }

    return glide;
}
```
初始化glide单例对象并返回

## Glide的checkAndInitializeGlide
```java
    private static void checkAndInitializeGlide(Context context) {
    // In the thread running initGlide(), one or more classes may call Glide.get(context).
    // Without this check, those calls could trigger infinite recursion.
    if (isInitializing) {
      throw new IllegalStateException("You cannot call Glide.get() in registerComponents(),"
          + " use the provided Glide instance instead");
    }
    isInitializing = true;
    initializeGlide(context);
    isInitializing = false;
  }
```
调用initializeGlide进行初始化

## Glide的initializeGlide
```java
    private static void initializeGlide(Context context) {
    Context applicationContext = context.getApplicationContext();

    // 没有自定义GlideModule所以annotationGeneratedModule为null
    GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();

    List<GlideModule> manifestModules = Collections.emptyList();
    if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
      // annotationGeneratedModule为null执行该处，由于manifest.xml中也没有定义
      // GlideModule，所以manifestModules是一个size为0的list
      manifestModules = new ManifestParser(applicationContext).parse();
    }

    // annotationGeneratedModule为null这里显然都不执行
    if (annotationGeneratedModule != null
        && !annotationGeneratedModule.getExcludedModuleClasses().isEmpty()) {
      Set<Class<?>> excludedModuleClasses =
          annotationGeneratedModule.getExcludedModuleClasses();
      for (Iterator<GlideModule> iterator = manifestModules.iterator(); iterator.hasNext();) {
        GlideModule current = iterator.next();
        if (!excludedModuleClasses.contains(current.getClass())) {
          continue;
        }
        if (Log.isLoggable(TAG, Log.DEBUG)) {
          Log.d(TAG, "AppGlideModule excludes manifest GlideModule: " + current);
        }
        iterator.remove();
      }
    }

    ...log 相关

    // annotationGeneratedModule为null -> factory为null
    RequestManagerRetriever.RequestManagerFactory factory =
        annotationGeneratedModule != null
            ? annotationGeneratedModule.getRequestManagerFactory() : null;
    // setRequestManagerFactory中传入的factory为null
    GlideBuilder builder = new GlideBuilder()
        .setRequestManagerFactory(factory);
    
    ...忽略

    // build方法初始化glide,会调用RequestManagerRetriever requestManagerRetriever 
    // = new RequestManagerRetriever(requestManagerFactory);requestManagerFactory为null，
    // 使用DEFAULT_FACTORY
    Glide glide = builder.build(applicationContext);
    
    ...忽略

    Glide.glide = glide;
  }
```
该方法就是glide真正初始化的地方

## Glide的getRequestManagerRetriever
```java
  public RequestManagerRetriever getRequestManagerRetriever() {
    return requestManagerRetriever;
  }
```
返回glide的requestManagerRetriever对象，在GlideBuilder的build方法生成glide对象时已经赋值

## RequestManagerRetriever的get
```java
  // 该方法有多个重载
  public RequestManager get(FragmentActivity activity) {
    // 如果不在主线程，则调用Context为参数的get
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      FragmentManager fm = activity.getSupportFragmentManager();
      return supportFragmentGet(activity, fm, null /*parentHint*/);
    }
  }
  // Context为参数的get
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

    // context是Application
    return getApplicationManager(context);
  }
```
该方法返回的RequestManager对象就是Glide的with方法返回的RequestManager对象

## RequestManagerRetriever的supportFragmentGet
```java
private RequestManager supportFragmentGet(Context context, FragmentManager fm,
      Fragment parentHint) {
    // 生成了一个SupportRequestManagerFragment的对象，并调用了FragmentManager的add方法
    SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint);
    RequestManager requestManager = current.getRequestManager();
    // 设置SupportRequestManagerFragment对象的requestManager
    if (requestManager == null) {
      // 获取gilde单例
      Glide glide = Glide.get(context);
      // factory就是DEFAULT_FACTORY
      requestManager =
          factory.build(glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode());
      current.setRequestManager(requestManager);
    }
    return requestManager;
}
```

## RequestManagerRetriever的getApplicationManager
```java
  private RequestManager getApplicationManager(Context context) {
    // Either an application context or we're on a background thread.
    if (applicationManager == null) {
      synchronized (this) {
        if (applicationManager == null) {
          // Normally pause/resume is taken care of by the fragment we add to the fragment or
          // activity. However, in this case since the manager attached to the application will not
          // receive lifecycle events, we must force the manager to start resumed using
          // ApplicationLifecycle.

          // TODO(b/27524013): Factor out this Glide.get() call.
          Glide glide = Glide.get(context);
          applicationManager =
              factory.build(glide, new ApplicationLifecycle(), new EmptyRequestManagerTreeNode());
        }
      }
    }

    return applicationManager;
  }
```
初始化单例对象applicationManager并返回

## RequestManager的load
```java
  public RequestBuilder<Drawable> load(@Nullable Object model) {
    return asDrawable().load(model);
  }
```

## RequestManager的asDrawable
```java
  public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class).transition(new DrawableTransitionOptions());
  }

  // RequestManager的as
  public <ResourceType> RequestBuilder<ResourceType> as(Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass);
  }

  // RequestBuilder的transition
  public RequestBuilder<TranscodeType> transition(
      @NonNull TransitionOptions<?, ? super TranscodeType> transitionOptions) {
    this.transitionOptions = Preconditions.checkNotNull(transitionOptions);
    return this;
  }
```
初始化一个RequestBuilder并设置其TransitionOptions

## RequestBuilder的load
```java
  // 该函数有很多重载，只看String作为参数的重载
  public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
  }
```

## RequestBuilder的loadGeneric
```java
  private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    this.model = model;
    isModelSet = true;
    return this;
  }
```

## RequestBuilder的into(ImageView view)
```java
  public Target<TranscodeType> into(ImageView view) {
    // 检查是否是主线程，view是否为null
    Util.assertMainThread();
    Preconditions.checkNotNull(view);

    // requestOptions是glide加载的一些选项
    if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      if (requestOptions.isLocked()) {
        requestOptions = requestOptions.clone();
      }
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions.optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions.optionalCenterInside();
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          requestOptions.optionalFitCenter();
          break;
        case FIT_XY:
          requestOptions.optionalCenterInside();
          break;
        case CENTER:
        case MATRIX:
        default:
          // Do nothing.
      }
    }

    // transcodeClass是Drawable.class
    return into(context.buildImageViewTarget(view, transcodeClass));
  }
```

## GlideContext的buildImageViewTarget
```java
  public <X> Target<X> buildImageViewTarget(ImageView imageView, Class<X> transcodeClass) {
    return imageViewTargetFactory.buildTarget(imageView, transcodeClass);
  }
```

## ImageViewTargetFactory的buildTarget
```java
  public <Z> Target<Z> buildTarget(ImageView view, Class<Z> clazz) {
    if (Bitmap.class.equals(clazz)) {
      return (Target<Z>) new BitmapImageViewTarget(view);
    } else if (Drawable.class.isAssignableFrom(clazz)) {
      return (Target<Z>) new DrawableImageViewTarget(view);
    } else {
      throw new IllegalArgumentException(
          "Unhandled class: " + clazz + ", try .as*(Class).transcode(ResourceTranscoder)");
    }
  }
```
因为传入的是Drawable.class，所以返回一个DrawableImageViewTarget的对象

## RequestBuilder的into(@NonNull Y target)
```java
  public <Y extends Target<TranscodeType>> Y into(@NonNull Y target) {
    Util.assertMainThread();
    Preconditions.checkNotNull(target);

    if (!isModelSet) {
      throw new IllegalArgumentException("You must call #load() before calling #into()");
    }

    // ViewTarget的getRequest，最后返回的是imageView中的getTag，没有地方设置，为null
    Request previous = target.getRequest();

    if (previous != null) {
      requestManager.clear(target);
    }

    /**
     * Throws if any further mutations are attempted.
     *
     * <p> Once locked, the only way to unlock is to use {@link #clone()} </p>
     */
    requestOptions.lock();

    Request request = buildRequest(target);

    // ViewTarget的setRequest，最后调用的是imageView中的setTag
    target.setRequest(request);

    requestManager.track(target, request);

    return target;
  }
```

## RequestBuilder的buildRequest，buildRequestRecursive
```java
  private Request buildRequest(Target<TranscodeType> target) {
    return buildRequestRecursive(target, null, transitionOptions, requestOptions.getPriority(),
        requestOptions.getOverrideWidth(), requestOptions.getOverrideHeight());
  }

  private Request buildRequestRecursive(Target<TranscodeType> target,
      @Nullable ThumbnailRequestCoordinator parentCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority, int overrideWidth, int overrideHeight) {
    if (thumbnailBuilder != null) {
      // Recursive case: contains a potentially recursive thumbnail request builder.
      if (isThumbnailBuilt) {
        throw new IllegalStateException("You cannot use a request as both the main request and a "
            + "thumbnail, consider using clone() on the request(s) passed to thumbnail()");
      }

      TransitionOptions<?, ? super TranscodeType> thumbTransitionOptions =
          thumbnailBuilder.transitionOptions;
      if (DEFAULT_ANIMATION_OPTIONS.equals(thumbTransitionOptions)) {
        thumbTransitionOptions = transitionOptions;
      }

      Priority thumbPriority = thumbnailBuilder.requestOptions.isPrioritySet()
          ? thumbnailBuilder.requestOptions.getPriority() : getThumbnailPriority(priority);

      int thumbOverrideWidth = thumbnailBuilder.requestOptions.getOverrideWidth();
      int thumbOverrideHeight = thumbnailBuilder.requestOptions.getOverrideHeight();
      if (Util.isValidDimensions(overrideWidth, overrideHeight)
          && !thumbnailBuilder.requestOptions.isValidOverride()) {
        thumbOverrideWidth = requestOptions.getOverrideWidth();
        thumbOverrideHeight = requestOptions.getOverrideHeight();
      }

      ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
      Request fullRequest = obtainRequest(target, requestOptions, coordinator,
          transitionOptions, priority, overrideWidth, overrideHeight);
      isThumbnailBuilt = true;
      // Recursively generate thumbnail requests.
      Request thumbRequest = thumbnailBuilder.buildRequestRecursive(target, coordinator,
          thumbTransitionOptions, thumbPriority, thumbOverrideWidth, thumbOverrideHeight);
      isThumbnailBuilt = false;
      coordinator.setRequests(fullRequest, thumbRequest);
      return coordinator;
    } else if (thumbSizeMultiplier != null) {
      // Base case: thumbnail multiplier generates a thumbnail request, but cannot recurse.
      ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
      Request fullRequest = obtainRequest(target, requestOptions, coordinator, transitionOptions,
          priority, overrideWidth, overrideHeight);
      RequestOptions thumbnailOptions = requestOptions.clone()
          .sizeMultiplier(thumbSizeMultiplier);

      Request thumbnailRequest = obtainRequest(target, thumbnailOptions, coordinator,
          transitionOptions, getThumbnailPriority(priority), overrideWidth, overrideHeight);

      coordinator.setRequests(fullRequest, thumbnailRequest);
      return coordinator;
    } else {
      // 目前只需要看obtainRequest
      return obtainRequest(target, requestOptions, parentCoordinator, transitionOptions, priority,
          overrideWidth, overrideHeight);
    }
  }
```

## RequestBuilder的obtainRequest
```java
  private Request obtainRequest(Target<TranscodeType> target,
      RequestOptions requestOptions, RequestCoordinator requestCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions, Priority priority,
      int overrideWidth, int overrideHeight) {
    requestOptions.lock();

    return SingleRequest.obtain(
        context,
        model,
        transcodeClass,
        requestOptions,
        overrideWidth,
        overrideHeight,
        priority,
        target,
        requestListener,
        requestCoordinator,
        context.getEngine(),
        transitionOptions.getTransitionFactory());
  }
```

## SingleRequest的obtain
```java
  public static <R> SingleRequest<R> obtain(
      GlideContext glideContext,
      Object model,
      Class<R> transcodeClass,
      RequestOptions requestOptions,
      int overrideWidth,
      int overrideHeight,
      Priority priority,
      Target<R> target,
      RequestListener<R> requestListener,
      RequestCoordinator requestCoordinator,
      Engine engine,
      TransitionFactory<? super R> animationFactory) {
        // 先尝试从POOL中取一个request，否则new一个
    @SuppressWarnings("unchecked") SingleRequest<R> request =
        (SingleRequest<R>) POOL.acquire();
    if (request == null) {
      request = new SingleRequest<>();
    }

    // 初始化request的数据
    request.init(
        glideContext,
        model,
        transcodeClass,
        requestOptions,
        overrideWidth,
        overrideHeight,
        priority,
        target,
        requestListener,
        requestCoordinator,
        engine,
        animationFactory);
    return request;
  }
```
终于拿到了request对象

## RequestManager的track
```java
  void track(Target<?> target, Request request) {
    targetTracker.track(target);
    requestTracker.runRequest(request);
  }
```

## TargetTracker的track
```java
  public void track(Target<?> target) {
    targets.add(target);
  }
```
把target加入targets中，targets是一个Set对象

## RequestTracker的runRequest
```java
  public void runRequest(Request request) {
    // requests是一个Set对象
    requests.add(request);
    // isPaused为false，调用request的begin，否则将request加入pendingRequests
    if (!isPaused) {
      request.begin();
    } else {
      pendingRequests.add(request);
    }
  }
```

## SingleRequest的begin
```java
  @Override
  public void begin() {
    stateVerifier.throwIfRecycled();
    startTime = LogTime.getLogTime();
    // 显然model不为null，先不管这里
    if (model == null) {
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        width = overrideWidth;
        height = overrideHeight;
      }
      // Only log at more verbose log levels if the user has set a fallback drawable, because
      // fallback Drawables indicate the user expects null models occasionally.
      int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
      onLoadFailed(new GlideException("Received null model"), logLevel);
      return;
    }

    status = Status.WAITING_FOR_SIZE;
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
      onSizeReady(overrideWidth, overrideHeight);
    } else {
      target.getSize(this);
    }

    // 设置placeholder
    if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
        && canNotifyStatusChanged()) {
      target.onLoadStarted(getPlaceholderDrawable());
    }
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logV("finished run method in " + LogTime.getElapsedMillis(startTime));
    }
  }
```
没有找到加载图片相关的代码。。。

## SingleRequest的onSizeReady
```java
  @Override
  public void onSizeReady(int width, int height) {
    stateVerifier.throwIfRecycled();
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logV("Got onSizeReady in " + LogTime.getElapsedMillis(startTime));
    }
    if (status != Status.WAITING_FOR_SIZE) {
      return;
    }
    status = Status.RUNNING;

    float sizeMultiplier = requestOptions.getSizeMultiplier();
    this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
    this.height = maybeApplySizeMultiplier(height, sizeMultiplier);

    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logV("finished setup for calling load in " + LogTime.getElapsedMillis(startTime));
    }
    loadStatus = engine.load(
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
        requestOptions.getOptions(),
        requestOptions.isMemoryCacheable(),
        requestOptions.getUseUnlimitedSourceGeneratorsPool(),
        requestOptions.getOnlyRetrieveFromCache(),
        this);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
    }
  }
```
计算width和height，调用了engine.load

## Engine的load
```java
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
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb) {
    Util.assertMainThread();
    long startTime = LogTime.getLogTime();

    // 获取该model相关的key
    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
        resourceClass, transcodeClass, options);

    // 通过key去缓存中取数据
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
      // 缓存有数据直接用缓存的数据，cb就是实现了ResourceCallback接口的SingleRequest对象
      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return null;
    }

    // 从activeResources中取数据，有就直接用
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return null;
    }

    // 没有缓存，则查看jobs中是否有相应EngineJob，有的话为其设置ResourceCallback cb
    // 并返回LoadStatus对象
    EngineJob<?> current = jobs.get(key);
    if (current != null) {
      current.addCallback(cb);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Added to existing load", startTime, key);
      }
      return new LoadStatus(cb, current);
    }

    // jobs中没有相应EngineJob，重新构建一个并将其加入jobs，调用其start方法执行DecodeJob，返回LoadStatus对象
    EngineJob<R> engineJob = engineJobFactory.build(key, isMemoryCacheable,
        useUnlimitedSourceExecutorPool);
    DecodeJob<R> decodeJob = decodeJobFactory.build(
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
        onlyRetrieveFromCache,
        options,
        engineJob);
    jobs.put(key, engineJob);
    engineJob.addCallback(cb);
    engineJob.start(decodeJob);

    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
  }
```

## EngineJob的start
```java
  public void start(DecodeJob<R> decodeJob) {
    this.decodeJob = decodeJob;
    GlideExecutor executor = decodeJob.willDecodeFromCache()
        ? diskCacheExecutor
        : getActiveSourceExecutor();
    executor.execute(decodeJob);
  }
```
GlideExecutor是一个ThreadPoolExecutor，decodeJob是实现了Runnbale接口的对象，

## DecodeJob的run
```java
  @Override
  public void run() {
    // This should be much more fine grained, but since Java's thread pool implementation silently
    // swallows all otherwise fatal exceptions, this will at least make it obvious to developers
    // that something is failing.
    TraceCompat.beginSection("DecodeJob#run");
    try {
      if (isCancelled) {
        notifyFailed();
        return;
      }
      runWrapped();
    } catch (RuntimeException e) {
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(TAG, "DecodeJob threw unexpectedly"
            + ", isCancelled: " + isCancelled
            + ", stage: " + stage, e);
      }
      // When we're encoding we've already notified our callback and it isn't safe to do so again.
      if (stage != Stage.ENCODE) {
        notifyFailed();
      }
      if (!isCancelled) {
        throw e;
      }
    } finally {
      if (currentFetcher != null) {
        currentFetcher.cleanup();
      }
      TraceCompat.endSection();
    }
  }
```
继续看DecodeJob得runWrapped

## DecodeJob的runWrapped
```java
  private void runWrapped() {
    // runReason为RunReason.INITIALIZE
     switch (runReason) {
      case INITIALIZE:
        stage = getNextStage(Stage.INITIALIZE);
        // 没有任何缓存的情况下获得SourceGenerator
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

## DecodeJob的runGenerators
```java
  private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    while (!isCancelled && currentGenerator != null
        && !(isStarted = currentGenerator.startNext())) {
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

    // Otherwise a generator started a new load and we expect to be called back in
    // onDataFetcherReady.
  }
```

## DecodeJob的getNextStage
```java
  private Stage getNextStage(Stage current) {
    switch (current) {
      case INITIALIZE:
        return diskCacheStrategy.decodeCachedResource()
            ? Stage.RESOURCE_CACHE : getNextStage(Stage.RESOURCE_CACHE);
      case RESOURCE_CACHE:
        return diskCacheStrategy.decodeCachedData()
            ? Stage.DATA_CACHE : getNextStage(Stage.DATA_CACHE);
      case DATA_CACHE:
        // Skip loading from source if the user opted to only retrieve the resource from cache.
        return onlyRetrieveFromCache ? Stage.FINISHED : Stage.SOURCE;
      case SOURCE:
      case FINISHED:
        return Stage.FINISHED;
      default:
        throw new IllegalArgumentException("Unrecognized stage: " + current);
    }
  }
```

## DecodeJob的getNextGenerator
```java
  private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
      case RESOURCE_CACHE:
        return new ResourceCacheGenerator(decodeHelper, this);
      case DATA_CACHE:
        return new DataCacheGenerator(decodeHelper, this);
      case SOURCE:
        return new SourceGenerator(decodeHelper, this);
      case FINISHED:
        return null;
      default:
        throw new IllegalStateException("Unrecognized stage: " + stage);
    }
  }
```
现在只关心SourceGenerator

## SourceGenerator的startNext
```java
  @Override
  public boolean startNext() {
    if (dataToCache != null) {
      Object data = dataToCache;
      dataToCache = null;
      cacheData(data);
    }

    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
      return true;
    }
    sourceCacheGenerator = null;

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      // 通过DecodeHelper获取一个loadData
      loadData = helper.getLoadData().get(loadDataListIndex++);
      if (loadData != null
          && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
          || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
        started = true;
        // loadData.fetcher是HttpUrlFetcher对象
        loadData.fetcher.loadData(helper.getPriority(), this);
      }
    }
    return started;
  }
```

## HttpUrlFetcher的loadData
```java
  @Override
  public void loadData(Priority priority, DataCallback<? super InputStream> callback) {
    long startTime = LogTime.getLogTime();
    final InputStream result;
    try {
      // HttpUrlFetcher的loadDataWithRedirects执行网络请求
      result = loadDataWithRedirects(glideUrl.toURL(), 0 /*redirects*/, null /*lastUrl*/,
          glideUrl.getHeaders());
    } catch (IOException e) {
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(TAG, "Failed to load data for url", e);
      }
      callback.onLoadFailed(e);
      return;
    }

    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(TAG, "Finished http url fetcher fetch in " + LogTime.getElapsedMillis(startTime)
          + " ms and loaded " + result);
    }
    // 这里最终调用到了DecodeJob的onDataFetcherReady
    callback.onDataReady(result);
  }
```

## DecodeJob的onDataFetcherReady
```java
  @Override
  public void onDataFetcherReady(Key sourceKey, Object data, DataFetcher<?> fetcher,
      DataSource dataSource, Key attemptedKey) {
    this.currentSourceKey = sourceKey;
    this.currentData = data;
    this.currentFetcher = fetcher;
    this.currentDataSource = dataSource;
    this.currentAttemptingKey = attemptedKey;
    if (Thread.currentThread() != currentThread) {
      runReason = RunReason.DECODE_DATA;
      callback.reschedule(this);
    } else {
      TraceCompat.beginSection("DecodeJob.decodeFromRetrievedData");
      try {
        decodeFromRetrievedData();
      } finally {
        TraceCompat.endSection();
      }
    }
  }
```

## DecodeJob的decodeFromRetrievedData
```java
  private void decodeFromRetrievedData() {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logWithTimeAndKey("Retrieved data", startFetchTime,
          "data: " + currentData
          + ", cache key: " + currentSourceKey
          + ", fetcher: " + currentFetcher);
    }
    Resource<R> resource = null;
    try {
      resource = decodeFromData(currentFetcher, currentData, currentDataSource);
    } catch (GlideException e) {
      e.setLoggingDetails(currentAttemptingKey, currentDataSource);
      exceptions.add(e);
    }
    if (resource != null) {
      notifyEncodeAndRelease(resource, currentDataSource);
    } else {
      runGenerators();
    }
  }
```

## DecodeJob的notifyEncodeAndRelease
```java
  private void notifyEncodeAndRelease(Resource<R> resource, DataSource dataSource) {
    if (resource instanceof Initializable) {
      ((Initializable) resource).initialize();
    }

    Resource<R> result = resource;
    LockedResource<R> lockedResource = null;
    if (deferredEncodeManager.hasResourceToEncode()) {
      lockedResource = LockedResource.obtain(resource);
      result = lockedResource;
    }

    // 此处调用到EngineJob的onResourceReady，最终通过
    notifyComplete(result, dataSource);

    stage = Stage.ENCODE;
    try {
      if (deferredEncodeManager.hasResourceToEncode()) {
        deferredEncodeManager.encode(diskCacheProvider, options);
      }
    } finally {
      if (lockedResource != null) {
        lockedResource.unlock();
      }
      onEncodeComplete();
    }
  }
```
notifyComplete调用到EngineJob的onResourceReady，最终通过Handler机制在主线程调用到DecodeJob的handleResultOnMainThread

## EngineJob的handleResultOnMainThread
```java
  void handleResultOnMainThread() {
    stateVerifier.throwIfRecycled();
    if (isCancelled) {
      resource.recycle();
      release(false /*isRemovedFromQueue*/);
      return;
    } else if (cbs.isEmpty()) {
      throw new IllegalStateException("Received a resource without any callbacks to notify");
    } else if (hasResource) {
      throw new IllegalStateException("Already have resource");
    }
    engineResource = engineResourceFactory.build(resource, isCacheable);
    hasResource = true;

    // Hold on to resource for duration of request so we don't recycle it in the middle of
    // notifying if it synchronously released by one of the callbacks.
    engineResource.acquire();
    // 这个listener就是Engine对象
    listener.onEngineJobComplete(key, engineResource);

    // 这里会回调SingleRequest的onResourceReady
    for (ResourceCallback cb : cbs) {
      if (!isInIgnoredCallbacks(cb)) {
        engineResource.acquire();
        cb.onResourceReady(engineResource, dataSource);
      }
    }
    // Our request is complete, so we can release the resource.
    engineResource.release();

    release(false /*isRemovedFromQueue*/);
  }
```

## Engine的onEngineJobComplete
```java
  @SuppressWarnings("unchecked")
  @Override
  public void onEngineJobComplete(Key key, EngineResource<?> resource) {
    Util.assertMainThread();
    // A null resource indicates that the load failed, usually due to an exception.
    if (resource != null) {
      resource.setResourceListener(key, this);

      if (resource.isCacheable()) {
        activeResources.put(key, new ResourceWeakReference(key, resource, getReferenceQueue()));
      }
    }
    // TODO: should this check that the engine job is still current?
    jobs.remove(key);
  }
```

## SingleRequest的onResourceReady
```java
  /**
   * A callback method that should never be invoked directly.
   */
  @SuppressWarnings("unchecked")
  @Override
  public void onResourceReady(Resource<?> resource, DataSource dataSource) {
    stateVerifier.throwIfRecycled();
    loadStatus = null;
    if (resource == null) {
      GlideException exception = new GlideException("Expected to receive a Resource<R> with an "
          + "object of " + transcodeClass + " inside, but instead got null.");
      onLoadFailed(exception);
      return;
    }

    Object received = resource.get();
    if (received == null || !transcodeClass.isAssignableFrom(received.getClass())) {
      releaseResource(resource);
      GlideException exception = new GlideException("Expected to receive an object of "
          + transcodeClass + " but instead" + " got "
          + (received != null ? received.getClass() : "") + "{" + received + "} inside" + " "
          + "Resource{" + resource + "}."
          + (received != null ? "" : " " + "To indicate failure return a null Resource "
          + "object, rather than a Resource object containing null data."));
      onLoadFailed(exception);
      return;
    }

    if (!canSetResource()) {
      releaseResource(resource);
      // We can't put the status to complete before asking canSetResource().
      status = Status.COMPLETE;
      return;
    }

    onResourceReady((Resource<R>) resource, (R) received, dataSource);
  }

  /**
   * Internal {@link #onResourceReady(Resource, DataSource)} where arguments are known to be safe.
   *
   * @param resource original {@link Resource}, never <code>null</code>
   * @param result   object returned by {@link Resource#get()}, checked for type and never
   *                 <code>null</code>
   */
  private void onResourceReady(Resource<R> resource, R result, DataSource dataSource) {
    // We must call isFirstReadyResource before setting status.
    boolean isFirstResource = isFirstReadyResource();
    status = Status.COMPLETE;
    this.resource = resource;

    if (glideContext.getLogLevel() <= Log.DEBUG) {
      Log.d(GLIDE_TAG, "Finished loading " + result.getClass().getSimpleName() + " from "
          + dataSource + " for " + model + " with size [" + width + "x" + height + "] in "
          + LogTime.getElapsedMillis(startTime) + " ms");
    }

    if (requestListener == null
        || !requestListener.onResourceReady(result, model, target, dataSource, isFirstResource)) {
      Transition<? super R> animation =
          animationFactory.build(dataSource, isFirstResource);
      // 此处调用ImageViewTarget的onResourceReady
      target.onResourceReady(result, animation);
    }

    notifyLoadSuccess();
  }
```

## ImageViewTarget的onResourceReady
```java
  @Override
  public void onResourceReady(Z resource, @Nullable Transition<? super Z> transition) {
    if (transition == null || !transition.transition(resource, this)) {
      setResourceInternal(resource);
    } else {
      maybeUpdateAnimatable(resource);
    }
  }

  private void setResourceInternal(@Nullable Z resource) {
    maybeUpdateAnimatable(resource);
    setResource(resource);
  }

  // DrawableImageViewTarget的setResource
  @Override
  protected void setResource(@Nullable Drawable resource) {
    view.setImageDrawable(resource);
  }
```
在setResourceInternal中设置ImageView的资源

## DecodeJob的reschedule
```java
  @Override
  public void reschedule() {
    runReason = RunReason.SWITCH_TO_SOURCE_SERVICE;
    // 这个callback就是上面的EngineJob的对象
    callback.reschedule(this);
  }
```

## EngineJob的reschedule
```java
  @Override
  public void reschedule(DecodeJob<?> job) {
    // 得到一个GlideExecutor对象，继续执行DecodeJob对象
    // 又会执行到runWrapped，
    getActiveSourceExecutor().execute(job);
  }
```



