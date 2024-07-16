---
title: gilde源码分析(1)-整体流程
date: 2024-07-16 22:34:41
tags: [Android,glide,源码分析]
categories: Android三方库源码分析
---
### 概述

本篇是该博客的一篇技术文章，也是安卓三方库源码分析的第一篇文章。我们首先来分析的库是图片库--glide.哪怕是安卓开发的入门者都熟悉glide的使用方法。

```java
Glide.with(context).load(url).into(imageView)
```

这里我们就可以看出这个库的设计之精巧，很明显这个库使用了builder的设计模式，使用起来十分简单方便。我们就从这个从网络url获取图片的例子开始，挨个分析这里的三个方法。分析完毕之后就会发现在这个简单的api背后，隐藏了负责的流程。

### 从with开始

```java
  //with方法其实是最外的一个封装，内部执行了两个方法，getRetriever和get
  public static RequestManager with(@NonNull Context context) {
    return getRetriever(context).get(context);
  }
  //这里还有很多层嵌套，就不一一展开，感兴趣可以看源码
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    // Context could be null for other reasons (ie the user passes in null), but in practice it will
    // only occur due to errors with the Fragment lifecycle.
    Preconditions.checkNotNull(context, DESTROYED_ACTIVITY_WARNING);
    return Glide.get(context).getRequestManagerRetriever();
  }

```
<!--more-->
Glide.get() 作用是获取一个glide实例，是同步调用；如果这个实例未初始化，会最终调用到Glide的build方法，触发初始化任务。

```java
  @NonNull
  Glide build(
      @NonNull Context context,
      List<GlideModule> manifestModules,
      AppGlideModule annotationGeneratedGlideModule) {
    if (sourceExecutor == null) {
    //资源执行器，线程池
      sourceExecutor = GlideExecutor.newSourceExecutor();
    }

    if (diskCacheExecutor == null) {
    //缓存执行器，线程池
      diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
    }

    if (animationExecutor == null) {
    //动画执行器，线程池
      animationExecutor = GlideExecutor.newAnimationExecutor();
    }

    if (memorySizeCalculator == null) {
      memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
    }

    if (connectivityMonitorFactory == null) {
      connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
    }

    if (bitmapPool == null) {
      int size = memorySizeCalculator.getBitmapPoolSize();
      if (size > 0) {
        bitmapPool = new LruBitmapPool(size);
      } else {
        bitmapPool = new BitmapPoolAdapter();
      }
    }

    if (arrayPool == null) {
      arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
    }
    //初始化内存缓存
    if (memoryCache == null) {
      memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }
    //初始化磁盘缓存
    if (diskCacheFactory == null) {
      diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }
    //这个engine也很重要
    if (engine == null) {
      engine =
          new Engine(
              memoryCache,
              diskCacheFactory,
              diskCacheExecutor,
              sourceExecutor,
              GlideExecutor.newUnlimitedSourceExecutor(),
              animationExecutor,
              isActiveResourceRetentionAllowed);
    }

    if (defaultRequestListeners == null) {
      defaultRequestListeners = Collections.emptyList();
    } else {
      defaultRequestListeners = Collections.unmodifiableList(defaultRequestListeners);
    }

    GlideExperiments experiments = glideExperimentsBuilder.build();
    RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory);
    //最终返回实例
    return new Glide(
        context,
        engine,
        memoryCache,
        bitmapPool,
        arrayPool,
        requestManagerRetriever,
        connectivityMonitorFactory,
        logLevel,
        defaultRequestOptionsFactory,
        defaultTransitionOptions,
        defaultRequestListeners,
        manifestModules,
        annotationGeneratedGlideModule,
        experiments);
  }
```

  RequestManagerRetriever中这里get方法的重载方法很多，但最终都是尝试获取到application的context，并利用他从而创建一个生命周期和application相同的requestManager并返回。

```java

    @NonNull
  public RequestManager get(@NonNull Context context) {
    if (context == null) {
      throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
      if (context instanceof FragmentActivity) {
        return get((FragmentActivity) context);
      } else if (context instanceof ContextWrapper
          // Only unwrap a ContextWrapper if the baseContext has a non-null application context.
          // Context#createPackageContext may return a Context without an Application instance,
          // in which case a ContextWrapper may be used to attach one.
          && ((ContextWrapper) context).getBaseContext().getApplicationContext() != null) {
        return get(((ContextWrapper) context).getBaseContext());
      }
    }

    return getApplicationManager(context);
  }
```

### load方法

这里只是简单的介绍一下load方法的过程，其中针对不同类型的资源都有不同的重载，具体可以看源码很复杂。简单插一句，在大多数三方库里，都依靠多态，接口来实现功能，java基础不好的同学可以先做了解，对提高代码设计能力很有帮助。

```java
  public RequestBuilder<Drawable> load(@Nullable String string) {
   //先不分析具体实践，这里就是通过asDrawable方法获取到了requestBuilder
    return asDrawable().load(string);
  }
    public RequestBuilder<Drawable> asDrawable() {
      //这里通过传入class进行区分
    return as(Drawable.class);
  }
    public <ResourceType> RequestBuilder<ResourceType> as(
      @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
  }
  
  //RequestBuilder#load
  
    public RequestBuilder<TranscodeType> load(@Nullable Bitmap bitmap) {
    return loadGeneric(bitmap).apply(diskCacheStrategyOf(DiskCacheStrategy.NONE));
  }
  
```

### into方法

整个流程中最复杂的一个方法，这里实现了图片资源的获取，解码，裁切等步骤，并通过handler机制将事件通知的主线程。

```java
  private <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,
      Executor callbackExecutor) {
    Preconditions.checkNotNull(target);
    if (!isModelSet) {
      throw new IllegalArgumentException("You must call #load() before calling #into()");
    }

    Request request = buildRequest(target, targetListener, options, callbackExecutor);

    Request previous = target.getRequest();
    if (request.isEquivalentTo(previous)
        && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
      // If the request is completed, beginning again will ensure the result is re-delivered,
      // triggering RequestListeners and Targets. If the request is failed, beginning again will
      // restart the request, giving it another chance to complete. If the request is already
      // running, we can let it continue running without interruption.
      if (!Preconditions.checkNotNull(previous).isRunning()) {
        // Use the previous request rather than the new one to allow for optimizations like skipping
        // setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
        // that are done in the individual Request.
        previous.begin();
      }
      return target;
    }

    requestManager.clear(target);
    target.setRequest(request);
    requestManager.track(target, request);

    return target;
  }
```

在into中通过一系列调用，使用engine触发异步加载

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
      boolean isScaleOnlyOrNoTransform,
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean useAnimationPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb,
      Executor callbackExecutor) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
    //创建任务的key，这个key用于缓存和资源匹配，很重要；很多图片库定制可以从这里入手
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
    //通过缓存找到资源
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);

      if (memoryResource == null) {
        return waitForExistingOrStartNewJob(
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
```

### 后续

glide代码走读预计会分为好几个文章，初步的规划为：

- 整体流程
- 缓存机制
- 初始化设置
- 部分解码器分析
- 线程模型
- 版本变迁

现在很多博客对于glide的讲解都停留在4.8.0，之后的版本有一些调整很多都没有提及，因为大多数互联网公司都没有使用新的技术。但是这这样对于新接触安卓的同学不是很友好，让他们觉得安卓开发停滞不前。同时这个系列更像是笔记，记录我在阅读代码过程中的一些关键点，很多地方没有写的很细。因我时间和经历有限，做不到教材的详细解答，所以大家如果有问题可以留言讨论，或者自行阅读源码。
ps：留言并未开通，我还在学习怎么设置留言~安啦。