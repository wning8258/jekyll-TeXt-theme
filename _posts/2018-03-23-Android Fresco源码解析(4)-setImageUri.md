---
layout: post
title: Android Fresco源码解析(4)-setImageUri
key: 20180323  Android Fresco源码解析(4)-setImageUri
tags: Android Fresco
typora-copy-images-to: ipic
---

# setImageUri

Fresco加载图片需要调用`SimpleDraweeView`的`setImageURI`方法

<!--more-->

```
  /**
   * Displays an image given by the uri.
   *
   * @param uri uri of the image
   * @param callerContext caller context
   */
  public void setImageURI(Uri uri, @Nullable Object callerContext) {
    DraweeController controller = mSimpleDraweeControllerBuilder  //第一步
        .setCallerContext(callerContext)
        .setUri(uri)
        .setOldController(getController())
        .build();  //第二步
    setController(controller);  //第三步
  }
```

PipelineDraweeControllerBuilder的继承关系如下：

![2684B59F-72D9-4560-B602-9679B76D9C0A](http://oon96myva.bkt.clouddn.com/md/nkqt4.png)

# 1.PipelineDraweeControllerBuilder初始化

先看一下`mSimpleDraweeControllerBuilder`变量，在上一节初始化解析时，

```
 private static void initializeDrawee(Context context) {
    sDraweeControllerBuilderSupplier = new PipelineDraweeControllerBuilderSupplier(context);
    SimpleDraweeView.initialize(sDraweeControllerBuilderSupplier);
  }
```

`PipelineDraweeControllerBuilderSupplier`作为参数传给了`SimpleDraweeView`的`initialize`方法，看一下`SimpleDraweeView`类里做了什么：

```
public class SimpleDraweeView extends GenericDraweeView {
  private SimpleDraweeControllerBuilder mSimpleDraweeControllerBuilder;

  private static Supplier<? extends SimpleDraweeControllerBuilder> sDraweeControllerBuilderSupplier;

  /** Initializes {@link SimpleDraweeView} with supplier of Drawee controller builders. */
  public static void initialize(
      Supplier<? extends SimpleDraweeControllerBuilder> draweeControllerBuilderSupplier) {
    sDraweeControllerBuilderSupplier = draweeControllerBuilderSupplier;
  }
  ......
   public SimpleDraweeView(Context context, AttributeSet attrs) {
    super(context, attrs);
    init();
  }
   private void init() {
   ......
    mSimpleDraweeControllerBuilder = sDraweeControllerBuilderSupplier.get();
  }
}
```



- 在`initialize`方法里，把其赋给了成员变量sDraweeControllerBuilderSupplier
- 在SimpleDraweeDraw的构造方法，调用`PipelineDraweeControllerBuilderSupplier`的`get`方法赋值给了`mSimpleDraweeControllerBuilder`，这里的`mSimpleDraweeControllerBuilder`类型是`PipelineDraweeControllerBuilder`，继承自`AbstractDraweeControllerBuilder`,而后者实现了`SimpleDraweeControllerBuilder`

我们先看一下`PipelineDraweeControllerBuilderSupplier`的`get()`方法做了啥：

```
  public PipelineDraweeControllerBuilderSupplier(
      Context context,
      ImagePipelineFactory imagePipelineFactory,
      Set<ControllerListener> boundControllerListeners) {
    mContext = context;
    mImagePipeline = imagePipelineFactory.getImagePipeline();
    mPipelineDraweeControllerFactory = new PipelineDraweeControllerFactory(
        context.getResources(),
        DeferredReleaser.getInstance(),
        imagePipelineFactory.getAnimatedDrawableFactory(),
        UiThreadImmediateExecutorService.getInstance()); //创建了mPipelineDraweeControllerFactory
    mBoundControllerListeners = boundControllerListeners;
  }

@Override
  public PipelineDraweeControllerBuilder get() {
    return new PipelineDraweeControllerBuilder(
        mContext,
        mPipelineDraweeControllerFactory,  //mPipelineDraweeController的工厂类，暂时不管
        mImagePipeline,  //ImagePipelineFactory工厂类返回的
        mBoundControllerListeners);
  }
```

这里我们需要注意，`PipelineDraweeControllerBuilderSupplier`实例化的时候，同时创建了`PipelineDraweeControllerFactory`类型的`mPipelineDraweeControllerFactory`,这是`PipelineDraweeController`的工厂类。

# 2. build()

`PipelineDraweeControllerBuilder`并没有实现`build()`方法，具体实现在父类`AbstractDraweeControllerBuilder`

中：

```
  /** Builds the specified controller. */
  @Override
  public AbstractDraweeController build() {
    validate();

    // if only a low-res request is specified, treat it as a final request.
    if (mImageRequest == null && mMultiImageRequests == null && mLowResImageRequest != null) {
      mImageRequest = mLowResImageRequest;
      mLowResImageRequest = null;
    }

    return buildController();  //调用了buldController()
  }

    /** Builds a regular controller. */
  protected AbstractDraweeController buildController() {
    AbstractDraweeController controller = obtainController();  //具体实现在obtainController这里
    controller.setRetainImageOnFailure(getRetainImageOnFailure());//失败时是否显示最后一张可见图片
    maybeBuildAndSetRetryManager(controller);  //是否点击重试
    maybeAttachListeners(controller);  //各种监听
    return controller;
  }

  /** Concrete builder classes should override this method to return a new controller. */
  protected abstract AbstractDraweeController obtainController();


  /**
   * Concrete builder classes should override this method to return a data source for the request.
   *
   * <p/>IMPORTANT: Do NOT ever call this method directly. This method is only to be called from
   * a supplier created in {#code getDataSourceSupplierForRequest(REQUEST, boolean)}.
   *
   * <p/>IMPORTANT: Make sure that you do NOT use any non-final field from this method, as the field
   * may change if the instance of this builder gets reused. If any such field is required, override
   * {#code getDataSourceSupplierForRequest(REQUEST, boolean)}, and store the field in a final
   * variable (same as it is done for callerContext).
   */
  protected abstract DataSource<IMAGE> getDataSourceForRequest(
      final REQUEST imageRequest,
      final Object callerContext,
      final boolean bitmapCacheOnly);

  /** Concrete builder classes should override this method to return {#code this}. */
  protected abstract BUILDER getThis();
```

`build()`—>`buildController()`—`>obtainController()`

buildcontroller()里增加了重试和监听一些逻辑，创建controller的逻辑在`obtainController`里，这个实现是由`PipelineDraweeControllerBuilder`做的：

```
 @Override
  protected PipelineDraweeController obtainController() {
    DraweeController oldController = getOldController();
    PipelineDraweeController controller;
    if (oldController instanceof PipelineDraweeController) {  //重用controller
      controller = (PipelineDraweeController) oldController;
      controller.initialize(
          obtainDataSourceSupplier(),
          generateUniqueControllerId(),
          getCallerContext());
    } else {
      controller = mPipelineDraweeControllerFactory.newController(
          obtainDataSourceSupplier(),
          generateUniqueControllerId(),
          getCallerContext());
    }
    return controller;
  }
```

我们比较关注的是12-15行这一段，首先是`obtainDataSourceSupplier`,其次是`mPipelineDraweeControllerFactory.newController`

## 1.obtainDataSourceSupplier

实现在父类`AbstractDraweeControllerBuilder`中：

```

  /** Gets the top-level data source supplier to be used by a controller. */
  protected Supplier<DataSource<IMAGE>> obtainDataSourceSupplier() {
    if (mDataSourceSupplier != null) {
      return mDataSourceSupplier;
    }

    Supplier<DataSource<IMAGE>> supplier = null;

    // final image supplier;
    if (mImageRequest != null) {  //单个图片请求
      supplier = getDataSourceSupplierForRequest(mImageRequest);
    } else if (mMultiImageRequests != null) {  //多个图片请求
      supplier = getFirstAvailableDataSourceSupplier(mMultiImageRequests, mTryCacheOnlyFirst);
    }

    // increasing-quality supplier; highest-quality supplier goes first
    if (supplier != null && mLowResImageRequest != null) {  //多种图片质量请求，高质量优先
      List<Supplier<DataSource<IMAGE>>> suppliers = new ArrayList<>(2);
      suppliers.add(supplier);
      suppliers.add(getDataSourceSupplierForRequest(mLowResImageRequest));
      supplier = IncreasingQualityDataSourceSupplier.create(suppliers);
    }

    // no image requests; use null data source supplier
    if (supplier == null) {
      supplier = DataSources.getFailedDataSourceSupplier(NO_REQUEST_EXCEPTION);
    }

    return supplier;
  }
```

这里的DataSource跟java的Future类似，具体可查看[Producer介绍](http://wning8258.com/2018/03/22/Android-Fresco%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90(2)-DataSource.html)

这里对多个图片请求，多种质量请求都进行了处理，我们最关心的是`getDataSourceSupplierForRequest`方法，

简单来说，其实就是获得一个订阅者（DataSource），这个DataSource用来返回图片请求的数据和状态。

`getDataSourceSupplierForRequest`方法：

```
/** Creates a data source supplier for the given image request. */
  protected Supplier<DataSource<IMAGE>> getDataSourceSupplierForRequest(REQUEST imageRequest) {
    return getDataSourceSupplierForRequest(imageRequest, /* bitmapCacheOnly */ false);
  }

  /** Creates a data source supplier for the given image request. */
  protected Supplier<DataSource<IMAGE>> getDataSourceSupplierForRequest(
      final REQUEST imageRequest,
      final boolean bitmapCacheOnly) {
    final Object callerContext = getCallerContext();
    return new Supplier<DataSource<IMAGE>>() {
      @Override
      public DataSource<IMAGE> get() {
        return getDataSourceForRequest(imageRequest, callerContext, bitmapCacheOnly);
      }
      @Override
      public String toString() {
        return Objects.toStringHelper(this)
            .add("request", imageRequest.toString())
            .toString();
      }
    };
  }
```

从11行，可以看出来，Supplier已经创建出来了，继续看`getDataSourceForRequest`方法。。。。

```
 /**
   * Concrete builder classes should override this method to return a data source for the request.
   *
   * <p/>IMPORTANT: Do NOT ever call this method directly. This method is only to be called from
   * a supplier created in {#code getDataSourceSupplierForRequest(REQUEST, boolean)}.
   *
   * <p/>IMPORTANT: Make sure that you do NOT use any non-final field from this method, as the field
   * may change if the instance of this builder gets reused. If any such field is required, override
   * {#code getDataSourceSupplierForRequest(REQUEST, boolean)}, and store the field in a final
   * variable (same as it is done for callerContext).
   */
  protected abstract DataSource<IMAGE> getDataSourceForRequest(
      final REQUEST imageRequest,
      final Object callerContext,
      final boolean bitmapCacheOnly);

```

…..

又是抽象的，去看他的实现（`PipelineDraweeControllerBuilder`）：

```
 @Override
  protected DataSource<CloseableReference<CloseableImage>> getDataSourceForRequest(
      ImageRequest imageRequest,
      Object callerContext,
      boolean bitmapCacheOnly) {
    if (bitmapCacheOnly) {
      return mImagePipeline.fetchImageFromBitmapCache(imageRequest, callerContext);
    } else {
      return mImagePipeline.fetchDecodedImage(imageRequest, callerContext);
    }
  }
```

emmmm...是不是感觉快出来点东西了，我们看到一个很重要的东西，`ImagePipeline`,之前说过，它就是用来加载图片的，看这两个方法：

- fetchImageFromBitmapCache 从缓存中加载bitmap
- fetchDecodedImage  取得解码后的图片

我们就看这个`fetchDecodedImage`喽：

```

  /**
   * Submits a request for execution and returns a DataSource representing the pending decoded
   * image(s).
   * <p>The returned DataSource must be closed once the client has finished with it.
   * @param imageRequest the request to submit
   * @return a DataSource representing the pending decoded image(s)
   */
  public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
      ImageRequest imageRequest,
      Object callerContext) {
    try {
      Producer<CloseableReference<CloseableImage>> producerSequence =
          mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);  //1
      return submitFetchRequest(  
          producerSequence,
          imageRequest,
          ImageRequest.RequestLevel.FULL_FETCH,
          callerContext); //2
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    }
  }
```

### 1.getDecodedImageProducerSequence

这里又出现的新的概念，`Producer`

`Produer`用于执行图片请求的各种任务，例如网络获取，硬盘缓存，内存缓存，解码，转换。通过`produceResults`把结果传递给`Consumer`,具体可查看上一篇文章：http://wning8258.com/2018/03/22/Android-Fresco%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90(3)-Producer&Consumer.html


我们这里先看一下` mProducerSequenceFactory.getDecodedImageProducerSequence`做了什么。

`mProducerSequenceFactory`是个什么？在哪创建的?

![](http://oon96myva.bkt.clouddn.com/md/cz7qa.png)

可以看到，在Fresco初始化的时候，`mProducerSequenceFactory`就被创建了。

ProducerSequence从字面意思来看，就是生产者序列，这里先列一下加载图片需要的Producer:

1. BitmapMemoryCacheGetProducer 只读内存缓存的producer
2. ThreadHandoffProducer 启动线程的producer，后续的producer都在线程中执行
3. BitmapMemoryCacheKeyMultiplexProducer 使用memory cache key合并请求的producer
4. BitmapMemoryCacheProducer 读取内存缓存的producer
5. DecodeProducer 解码图片的producer，渐进式JPEG图片，gif和webp等动画图片的解码
6. ResizeAndRotateProducer JPEG图片resizes和rotates处理
7. AddImageTransformMetaDataProducer 主要包装解码的consumer，并传递到下一个producer
8. EncodedCacheKeyMultiplexProducer 使用encoded cache key合并请求的producer
9. EncodedMemoryCacheProducer 读取未解码的内存缓存的producer
10. DiskCacheProducer 读取磁盘缓存的producer
11. WebpTranscodeProducer 包装转码WebP到JPEG/PNG的consumer，并传递到下一个producer
12. NetworkFetchProducer 网络请求的producer

fresco加载图片的时候，从1-12依次查找，如果某个Producer里

![E430B151-6E5F-4CBA-9076-AC0D8920E3FB](http://oon96myva.bkt.clouddn.com/md/4fjk4.png)

#### 1.三级缓存

1.Bitmap缓存

Bitmap缓存存储Bitmap对象，这些Bitmap对象可以立刻用来显示或者用于后处理

在5.0以下系统，Bitmap缓存位于ashmem，这样Bitmap对象的创建和释放将不会引发GC，更少的GC会使你的APP运行得更加流畅。

5.0及其以上系统，相比之下，内存管理有了很大改进，所以Bitmap缓存直接位于Java的heap上。

当应用在后台运行时，该内存会被清空。

2.未解码图片的内存缓存

这个缓存存储的是原始压缩格式的图片。从该缓存取到的图片在使用之前，需要先进行解码。

如果有调整大小，旋转，或者WebP编码转换工作需要完成，这些工作会在解码之前进行。

APP在后台时，这个缓存同样会被清空。

3.文件缓存

和未解码的内存缓存相似，文件缓存存储的是未解码的原始压缩格式的图片，在使用之前同样需要经过解码等处理。

#### 2.图片获取顺序

和内存缓存不一样，APP在后台时，内容是不会被清空的。即使关机也不会。用户可以随时用系统的设置菜单中进行清空缓存操作。 
**图片获取是由各级Producer实现的，而将获取到的图片添加到缓存中是由各级Cusumer来实现的。**

看一`mProducerSequenceFactory`的`getDecodedImageProducerSequence`方法：

```
  /**
   * Returns a sequence that can be used for a request for a decoded image.
   *
   * @param imageRequest the request that will be submitted
   * @return the sequence that should be used to process the request
   */
  public Producer<CloseableReference<CloseableImage>> getDecodedImageProducerSequence(
      ImageRequest imageRequest) {
    Producer<CloseableReference<CloseableImage>> pipelineSequence =
        getBasicDecodedImageSequence(imageRequest);
    if (imageRequest.getPostprocessor() != null) {
      return getPostprocessorSequence(pipelineSequence);
    } else {
      return pipelineSequence;
    }
  }
```

我们只关心`getBasicDecodedImageSequence`,先不考虑`getPostprocessorSequence`：

```
private Producer<CloseableReference<CloseableImage>> getBasicDecodedImageSequence(
      ImageRequest imageRequest) {
    Preconditions.checkNotNull(imageRequest);

    Uri uri = imageRequest.getSourceUri();
    Preconditions.checkNotNull(uri, "Uri is null.");
    if (UriUtil.isNetworkUri(uri)) {
      return getNetworkFetchSequence();
    } else if (UriUtil.isLocalFileUri(uri)) {
      if (MediaUtils.isVideo(MediaUtils.extractMime(uri.getPath()))) {
        return getLocalVideoFileFetchSequence();
      } else {
        return getLocalImageFileFetchSequence();
      }
    } else if (UriUtil.isLocalContentUri(uri)) {
      return getLocalContentUriFetchSequence();
    } else if (UriUtil.isLocalAssetUri(uri)) {
      return getLocalAssetFetchSequence();
    } else if (UriUtil.isLocalResourceUri(uri)) {
      return getLocalResourceFetchSequence();
    } else if (UriUtil.isDataUri(uri)) {
      return getDataFetchSequence();
    } else {
      String uriString = uri.toString();
      if (uriString.length() > 30) {
        uriString = uriString.substring(0, 30) + "...";
      }
      throw new RuntimeException("Unsupported uri scheme! Uri is: " + uriString);
    }
  }

```

这里可以看出，根据uri的不同类型会返回不同的sequence,我们经常用到的是网络请求，所以看一下`getNetworkFetchSequence`:

```
  /**
   * swallow result if prefetch -> bitmap cache get ->
   * background thread hand-off -> multiplex -> bitmap cache -> decode -> multiplex ->
   * encoded cache -> disk cache -> (webp transcode) -> network fetch.
   */
  private synchronized Producer<CloseableReference<CloseableImage>> getNetworkFetchSequence() {
    if (mNetworkFetchSequence == null) {
      mNetworkFetchSequence =
          newBitmapCacheGetToDecodeSequence(  //内存缓存解码
          getCommonNetworkFetchToEncodedMemorySequence());  //网络请求，编码到内存
    }
    return mNetworkFetchSequence;
  }
```

这里分为两部分：

##### 1.从BitmapCache到解码后的图片内存缓存序列（newBitmapCacheGetToDecodeSequence）

```
/**
   * Same as {@code newBitmapCacheGetToBitmapCacheSequence} but with an extra DecodeProducer.
   * @param inputProducer producer providing the input to the decode
   * @return bitmap cache get to decode sequence
   */
  private Producer<CloseableReference<CloseableImage>> newBitmapCacheGetToDecodeSequence(
      Producer<EncodedImage> inputProducer) {
    DecodeProducer decodeProducer = mProducerFactory.newDecodeProducer(inputProducer);  //DecodeProducer
    return newBitmapCacheGetToBitmapCacheSequence(decodeProducer);
  }
  
    /**
   * Bitmap cache get -> thread hand off -> multiplex -> bitmap cache
   * @param inputProducer producer providing the input to the bitmap cache
   * @return bitmap cache get to bitmap cache sequence
   */
  private Producer<CloseableReference<CloseableImage>> newBitmapCacheGetToBitmapCacheSequence(
      Producer<CloseableReference<CloseableImage>> inputProducer) {
    BitmapMemoryCacheProducer bitmapMemoryCacheProducer =
        mProducerFactory.newBitmapMemoryCacheProducer(inputProducer);  //BitmapMemoryCacheProducer
    BitmapMemoryCacheKeyMultiplexProducer bitmapKeyMultiplexProducer =  //
        mProducerFactory.newBitmapMemoryCacheKeyMultiplexProducer(bitmapMemoryCacheProducer);//BitmapMemoryCacheKeyMultiplexProducer
    ThreadHandoffProducer<CloseableReference<CloseableImage>> threadHandoffProducer =
        mProducerFactory.newBackgroundThreadHandoffProducer(
            bitmapKeyMultiplexProducer,
            mThreadHandoffProducerQueue); //ThreadHandoffProducer
    return mProducerFactory.newBitmapMemoryCacheGetProducer(threadHandoffProducer); //BitmapMemoryCacheGetProducer
  }
```

1. BitmapMemoryCacheGetProducer 只读内存缓存的producer(这个是Producer的起始点)

```
/**
 * Bitmap memory cache producer that is read-only.
 */
public class BitmapMemoryCacheGetProducer extends BitmapMemoryCacheProducer {

  @VisibleForTesting static final String PRODUCER_NAME = "BitmapMemoryCacheGetProducer";

  public BitmapMemoryCacheGetProducer(
      MemoryCache<CacheKey, CloseableImage> memoryCache,
      CacheKeyFactory cacheKeyFactory,
      Producer<CloseableReference<CloseableImage>> inputProducer) {
    super(memoryCache, cacheKeyFactory, inputProducer);
  }

  @Override
  protected Consumer<CloseableReference<CloseableImage>> wrapConsumer(
      final Consumer<CloseableReference<CloseableImage>> consumer,
      final CacheKey cacheKey) {
    // since this cache is read-only, we can pass our consumer directly to the next producer
    return consumer;
  }

  @Override
  protected String getProducerName() {
    return PRODUCER_NAME;
  }
}
```

2. ThreadHandoffProducer 启动线程的producer，后续的producer都在线程中执行(使用Executor

创建了新的线程)

3. BitmapMemoryCacheKeyMultiplexProducer 使用memory cache key合并请求的producer
4. BitmapMemoryCacheProducer 读取内存缓存的producer

**该类与BitmapMemoryCacheGetProducer的不同之处是，它在缓存中不存在数据时，会创建相应的Consumer，使用mMemoryCache.cache(cacheKey, newResult)将解压后的图片数据缓存到内存中去。**

```
/**
 * Memory cache producer for the bitmap memory cache.
 */
public class BitmapMemoryCacheProducer implements Producer<CloseableReference<CloseableImage>> {

  @VisibleForTesting static final String PRODUCER_NAME = "BitmapMemoryCacheProducer";
  @VisibleForTesting static final String VALUE_FOUND = "cached_value_found";

  private final MemoryCache<CacheKey, CloseableImage> mMemoryCache;
  private final CacheKeyFactory mCacheKeyFactory;
  private final Producer<CloseableReference<CloseableImage>> mInputProducer;

  public BitmapMemoryCacheProducer(
      MemoryCache<CacheKey, CloseableImage> memoryCache,
      CacheKeyFactory cacheKeyFactory,
      Producer<CloseableReference<CloseableImage>> inputProducer) {
    mMemoryCache = memoryCache;
    mCacheKeyFactory = cacheKeyFactory;
    mInputProducer = inputProducer;
  }

  @Override
  public void produceResults(
      final Consumer<CloseableReference<CloseableImage>> consumer,
      final ProducerContext producerContext) {

    final ProducerListener listener = producerContext.getListener();
    final String requestId = producerContext.getId();
    listener.onProducerStart(requestId, getProducerName());
    final ImageRequest imageRequest = producerContext.getImageRequest();
    final CacheKey cacheKey = mCacheKeyFactory.getBitmapCacheKey(imageRequest);

    CloseableReference<CloseableImage> cachedReference = mMemoryCache.get(cacheKey);

    if (cachedReference != null) {
      boolean isFinal = cachedReference.get().getQualityInfo().isOfFullQuality();
      if (isFinal) {
        listener.onProducerFinishWithSuccess(
            requestId,
            getProducerName(),
            listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "true") : null);
        consumer.onProgressUpdate(1f);
      }
      consumer.onNewResult(cachedReference, isFinal);
      cachedReference.close();
      if (isFinal) {
        return;
      }
    }

    if (producerContext.getLowestPermittedRequestLevel().getValue() >=
        ImageRequest.RequestLevel.BITMAP_MEMORY_CACHE.getValue()) {
      listener.onProducerFinishWithSuccess(
          requestId,
          getProducerName(),
          listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "false") : null);
      consumer.onNewResult(null, true);
      return;
    }

    Consumer<CloseableReference<CloseableImage>> wrappedConsumer = wrapConsumer(consumer, cacheKey);
    listener.onProducerFinishWithSuccess(
        requestId,
        getProducerName(),
        listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "false") : null);
    mInputProducer.produceResults(wrappedConsumer, producerContext);
  }

  protected Consumer<CloseableReference<CloseableImage>> wrapConsumer(
      final Consumer<CloseableReference<CloseableImage>> consumer,
      final CacheKey cacheKey) {
    return new DelegatingConsumer<
        CloseableReference<CloseableImage>,
        CloseableReference<CloseableImage>>(consumer) {
      @Override
      public void onNewResultImpl(CloseableReference<CloseableImage> newResult, boolean isLast) {
        // ignore invalid intermediate results and forward the null result if last
        if (newResult == null) {
          if (isLast) {
            getConsumer().onNewResult(null, true);
          }
          return;
        }
        // stateful results cannot be cached and are just forwarded
        if (newResult.get().isStateful()) {
          getConsumer().onNewResult(newResult, isLast);
          return;
        }
        // if the intermediate result is not of a better quality than the cached result,
        // forward the already cached result and don't cache the new result.
        if (!isLast) {
          CloseableReference<CloseableImage> currentCachedResult = mMemoryCache.get(cacheKey);
          if (currentCachedResult != null) {
            try {
              QualityInfo newInfo = newResult.get().getQualityInfo();
              QualityInfo cachedInfo = currentCachedResult.get().getQualityInfo();
              if (cachedInfo.isOfFullQuality() || cachedInfo.getQuality() >= newInfo.getQuality()) {
                getConsumer().onNewResult(currentCachedResult, false);
                return;
              }
            } finally {
              CloseableReference.closeSafely(currentCachedResult);
            }
          }
        }
        // cache and forward the new result
        CloseableReference<CloseableImage> newCachedResult =
            mMemoryCache.cache(cacheKey, newResult);   //这里把cache存储起来
        try {
          if (isLast) {
            getConsumer().onProgressUpdate(1f);
          }
          getConsumer().onNewResult(
              (newCachedResult != null) ? newCachedResult : newResult, isLast);
        } finally {
          CloseableReference.closeSafely(newCachedResult);
        }
      }
    };
  }
  ......
}

```

**重点在108行，根据cachekey和数据缓存在内存缓存里**

注意这里的32行`wrapConsumer`,使用了代理模式，每个producer都会调用consumer的方法,但是不同的producer需要在原有consumer的基础上处理自己的一些逻辑,这里呢?就需要将原来的consumer进行代理,调用时,先处理自己的逻辑,然后调用原有consumer的相关方法即可。

看一下`DelegatingConsumer`:

```
 */
public abstract class DelegatingConsumer<I, O> extends BaseConsumer<I> {

  private final Consumer<O> mConsumer;

  public DelegatingConsumer(Consumer<O> consumer) {
    mConsumer = consumer;
  }

  public Consumer<O> getConsumer() {
    return mConsumer;
  }

  @Override
  protected void onFailureImpl(Throwable t) {
    mConsumer.onFailure(t);
  }

  @Override
  protected void onCancellationImpl() {
    mConsumer.onCancellation();
  }

  @Override
  protected void onProgressUpdateImpl(float progress) {
    mConsumer.onProgressUpdate(progress);
  }
}
```

5. DecodeProducer 解码图片的producer，渐进式JPEG图片，gif和webp等动画图片的解码

##### 2.从网络请求到未解码的图片内存缓存序列（getCommonNetworkFetchToEncodedMemorySequence）

```

  /**
   * multiplex -> encoded cache -> disk cache -> (webp transcode) -> network fetch.
   */
  private synchronized Producer<EncodedImage> getCommonNetworkFetchToEncodedMemorySequence() {
    if (mCommonNetworkFetchToEncodedMemorySequence == null) {
      Producer<EncodedImage> inputProducer =
          newEncodedCacheMultiplexToTranscodeSequence(
              mProducerFactory.newNetworkFetchProducer(mNetworkFetcher));
      mCommonNetworkFetchToEncodedMemorySequence =
          ProducerFactory.newAddImageTransformMetaDataProducer(inputProducer);

      if (mResizeAndRotateEnabledForNetwork && !mDownsampleEnabled) {
        mCommonNetworkFetchToEncodedMemorySequence =
            mProducerFactory.newResizeAndRotateProducer(
                mCommonNetworkFetchToEncodedMemorySequence);
      }
    }
    return mCommonNetworkFetchToEncodedMemorySequence;
  }
```

6. ResizeAndRotateProducer JPEG图片resizes和rotates处理
7. AddImageTransformMetaDataProducer 主要包装解码的consumer，并传递到下一个producer
8. EncodedCacheKeyMultiplexProducer 使用encoded cache key合并请求的producer
9. EncodedMemoryCacheProducer 读取未解码的内存缓存的producer

**与BitmapMemoryCacheProducer类似，在缓存中不存在数据时，会创建相应的Consumer，使用 cachedResult = mMemoryCache.cache(cacheKey, ref);将图片数据缓存到未解码图片的内存缓存区中**

```
 @Override
  public void produceResults(
      final Consumer<EncodedImage> consumer,
      final ProducerContext producerContext) {

    final String requestId = producerContext.getId();
    final ProducerListener listener = producerContext.getListener();
    listener.onProducerStart(requestId, PRODUCER_NAME);
    final ImageRequest imageRequest = producerContext.getImageRequest();
    final CacheKey cacheKey = mCacheKeyFactory.getEncodedCacheKey(imageRequest);

    CloseableReference<PooledByteBuffer> cachedReference = mMemoryCache.get(cacheKey);
    try {
      if (cachedReference != null) {
        EncodedImage cachedEncodedImage = new EncodedImage(cachedReference);
        try {
          listener.onProducerFinishWithSuccess(
              requestId,
              PRODUCER_NAME,
              listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "true") : null);
          consumer.onProgressUpdate(1f);
          consumer.onNewResult(cachedEncodedImage, true);
          return;
        } finally {
          EncodedImage.closeSafely(cachedEncodedImage);
        }
      }

      if (producerContext.getLowestPermittedRequestLevel().getValue() >=
          ImageRequest.RequestLevel.ENCODED_MEMORY_CACHE.getValue()) {
        listener.onProducerFinishWithSuccess(
            requestId,
            PRODUCER_NAME,
            listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "false") : null);
        consumer.onNewResult(null, true);
        return;
      }

      Consumer<EncodedImage> consumerOfInputProducer = new DelegatingConsumer<
          EncodedImage,
          EncodedImage>(consumer) {
        @Override
        public void onNewResultImpl(EncodedImage newResult, boolean isLast) {
          // intermediate or null results are not cached, so we just forward them
          if (!isLast || newResult == null) {
            getConsumer().onNewResult(newResult, isLast);
            return;
          }
          // cache and forward the last result
          CloseableReference<PooledByteBuffer> ref = newResult.getByteBufferRef();
          if (ref != null) {
            CloseableReference<PooledByteBuffer> cachedResult;
            try {
              cachedResult = mMemoryCache.cache(cacheKey, ref);  //这里存储
            } finally {
              CloseableReference.closeSafely(ref);
            }
            if (cachedResult != null) {
              EncodedImage cachedEncodedImage;
              try {
                cachedEncodedImage = new EncodedImage(cachedResult);
                cachedEncodedImage.copyMetaDataFrom(newResult);
              } finally {
                CloseableReference.closeSafely(cachedResult);
              }
              try {
                getConsumer().onProgressUpdate(1f);
                getConsumer().onNewResult(cachedEncodedImage, true);
                return;
              } finally {
                EncodedImage.closeSafely(cachedEncodedImage);
              }
            }
          }
          getConsumer().onNewResult(newResult, true);
        }
      };
......
  }
```

10. DiskCacheProducer 读取磁盘缓存的producer

从disk缓存中获取数据，如果没有找到的话，使用NetworkFetchProducer获取数据并创建DiskCacheConsumer对象，将数据缓存到disk中

```
/**
   * Consumer that consumes results from next producer in the sequence.
   *
   * <p>The consumer puts the last result received into disk cache, and passes all results (success
   * or failure) down to the next consumer.
   */
  private class DiskCacheConsumer extends DelegatingConsumer<EncodedImage, EncodedImage> {

    private final BufferedDiskCache mCache;
    private final CacheKey mCacheKey;

    private DiskCacheConsumer(
        final Consumer<EncodedImage> consumer,
        final BufferedDiskCache cache,
        final CacheKey cacheKey) {
      super(consumer);
      mCache = cache;
      mCacheKey = cacheKey;
    }

    @Override
    public void onNewResultImpl(EncodedImage newResult, boolean isLast) {
      if (newResult != null && isLast) {
        mCache.put(mCacheKey, newResult);
      }
      getConsumer().onNewResult(newResult, isLast);
    }
  }
```

11. WebpTranscodeProducer 包装转码WebP到JPEG/PNG的consumer，并传递到下一个producer


12. NetworkFetchProducer 网络请求的producer

这里的网络请求是使用的httpurlconnection,当然可以自己配置，默认的配置是在ImagePipelineConfig中的

```
mNetworkFetcher =
        builder.mNetworkFetcher == null ?
            new HttpUrlConnectionNetworkFetcher() :
```

我们知道，在Fresco初始化的时候：

```
 /** Initializes {@link ImagePipelineFactory} with default config. */
  public static void initialize(Context context) {
    initialize(ImagePipelineConfig.newBuilder(context).build());
  }

  /** Initializes {@link ImagePipelineFactory} with the specified config. */
  public static void initialize(ImagePipelineConfig imagePipelineConfig) {
    sInstance = new ImagePipelineFactory(imagePipelineConfig);
  }
```

如果想自定义的话，使用`ImagePipelineConfig.newBuilder(context)`，创建出`ImagePipelineConfig.Builder`实例，该类中提供了对应的方法:

```
public Builder setNetworkFetcher(NetworkFetcher networkFetcher) {
      mNetworkFetcher = networkFetcher;
      return this;
    }
```

**不过需要注意的是：如果想使用okhttp,不能使用setNetworkFetcher方法，官网上也给了说明：**

For OkHttp2:

```
dependencies {
  // your project's other dependencies
  compile "com.facebook.fresco:imagepipeline-okhttp:0.12.0+"
}
```

For OkHttp3:

```
dependencies {
  // your project's other dependencies
  compile "com.facebook.fresco:imagepipeline-okhttp3:0.12.0+"
}
```

**配置Image pipeline这时也有一些不同，不再使用`ImagePipelineConfig.newBuilder`,而是使用`OkHttpImagePipelineConfigFactory`:**

```
Context context;
OkHttpClient okHttpClient; // build on your own
ImagePipelineConfig config = OkHttpImagePipelineConfigFactory
    .newBuilder(context, okHttpClient)
    . // other setters
    . // setNetworkFetcher is already called for you
    .build();
Fresco.initialize(context, config);
```

好了，我们知道图片加载肯定是从内存缓存开始的，也就是`BitmapMemoryCacheGetProducer`:

```
  @Override
  public void produceResults(
      final Consumer<CloseableReference<CloseableImage>> consumer,
      final ProducerContext producerContext) {

    final ProducerListener listener = producerContext.getListener();
    final String requestId = producerContext.getId();
    listener.onProducerStart(requestId, getProducerName());
    final ImageRequest imageRequest = producerContext.getImageRequest();
    final CacheKey cacheKey = mCacheKeyFactory.getBitmapCacheKey(imageRequest);

    CloseableReference<CloseableImage> cachedReference = mMemoryCache.get(cacheKey);

    if (cachedReference != null) {
      boolean isFinal = cachedReference.get().getQualityInfo().isOfFullQuality();
      if (isFinal) {
        listener.onProducerFinishWithSuccess(
            requestId,
            getProducerName(),
            listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "true") : null);
        consumer.onProgressUpdate(1f);
      }
      consumer.onNewResult(cachedReference, isFinal);
      cachedReference.close();
      if (isFinal) {
        return;
      }
    }

    if (producerContext.getLowestPermittedRequestLevel().getValue() >=
        ImageRequest.RequestLevel.BITMAP_MEMORY_CACHE.getValue()) {
      listener.onProducerFinishWithSuccess(
          requestId,
          getProducerName(),
          listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "false") : null);
      consumer.onNewResult(null, true);
      return;
    }

    Consumer<CloseableReference<CloseableImage>> wrappedConsumer = wrapConsumer(consumer, cacheKey);
    listener.onProducerFinishWithSuccess(
        requestId,
        getProducerName(),
        listener.requiresExtraMap(requestId) ? ImmutableMap.of(VALUE_FOUND, "false") : null);
    mInputProducer.produceResults(wrappedConsumer, producerContext);
  }
```

那么问题来了，它的`produceResults`方法，是被谁调用的呢？

我们回到`ImagePipeline`的`fetchDecodedImage`方法：

```

  /**
   * Submits a request for execution and returns a DataSource representing the pending decoded
   * image(s).
   * <p>The returned DataSource must be closed once the client has finished with it.
   * @param imageRequest the request to submit
   * @return a DataSource representing the pending decoded image(s)
   */
  public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
      ImageRequest imageRequest,
      Object callerContext) {
    try {
      Producer<CloseableReference<CloseableImage>> producerSequence =
          mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);  //1
      return submitFetchRequest(  
          producerSequence,
          imageRequest,
          ImageRequest.RequestLevel.FULL_FETCH,
          callerContext); //2
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    }
  }
```

我们上边一大串都只是看的`mProducerSequenceFactory.getDecodedImageProducerSequence`这一步，这里的`producerSequence`实际上就是`BitmapMemoryCacheGetProducer`,下一步就需要看看`submitFetchRequest`做了什么了。

### 2.submitFetchRequest

`submitFetchRequest`看字面意思就是提交请求，

```
 private <T> DataSource<CloseableReference<T>> submitFetchRequest(
      Producer<CloseableReference<T>> producerSequence,
      ImageRequest imageRequest,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit,
      Object callerContext) {
    try {
      ImageRequest.RequestLevel lowestPermittedRequestLevel =
          ImageRequest.RequestLevel.getMax(
              imageRequest.getLowestPermittedRequestLevel(),
              lowestPermittedRequestLevelOnSubmit);  //获取请求级别
      SettableProducerContext settableProducerContext = new SettableProducerContext(
          imageRequest,
          generateUniqueFutureId(),
          mRequestListener,
          callerContext,
          lowestPermittedRequestLevel,
        /* isPrefetch */ false,
          imageRequest.getProgressiveRenderingEnabled() ||
              !UriUtil.isNetworkUri(imageRequest.getSourceUri()),
          imageRequest.getPriority());
      return CloseableProducerToDataSourceAdapter.create(
          producerSequence,
          settableProducerContext,
          mRequestListener);  //创建datasource
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    }
```

1. 计算出当前图片最低的请求级别
2. 获得请求信息的上下文
3. 根据创建的SettableProducerContext，再利用Producer和DataSource的适配器，创建一个DataSource。

#### 1.最低请求级别

Image pipeline 加载图片时有一套明确的[请求流程](https://www.fresco-cn.org/docs/intro-image-pipeline.html)

1. 检查内存缓存，有如，立刻返回。这个操作是实时的。
2. 检查未解码的图片缓存，如有，解码并返回。
3. 检查磁盘缓存，如果有加载，解码，返回。
4. 下载或者加载本地文件。调整大小和旋转（如有），解码并返回。对于网络图来说，这一套流程下来是最耗时的。

`setLowestPermittedRequestLevel`允许设置一个最低请求级别，请求级别和上面对应地有以下几个取值:

- `BITMAP_MEMORY_CACHE`
- `ENCODED_MEMORY_CACHE`
- `DISK_CACHE`
- `FULL_FETCH`

如果你需要立即取到一个图片，或者在相对比较短时间内取到图片，否则就不显示的情况下，这非常有用。

#### 2.CloseableProducerToDataSourceAdapter

```

/**
 * DataSource<CloseableReference<T>> backed by a Producer<CloseableReference<T>>
 *
 * @param <T>
 */
@ThreadSafe
public class CloseableProducerToDataSourceAdapter<T>
    extends AbstractProducerToDataSourceAdapter<CloseableReference<T>> {

  public static <T> DataSource<CloseableReference<T>> create(
      Producer<CloseableReference<T>> producer,
      SettableProducerContext settableProducerContext,
      RequestListener listener) {
    return new CloseableProducerToDataSourceAdapter<T>(
        producer, settableProducerContext, listener);
  }

  private CloseableProducerToDataSourceAdapter(
      Producer<CloseableReference<T>> producer,
      SettableProducerContext settableProducerContext,
      RequestListener listener) {
    super(producer, settableProducerContext, listener);
  }

  @Override
  @Nullable
  public CloseableReference<T> getResult() {
    return CloseableReference.cloneOrNull(super.getResult());
  }

  @Override
  protected void closeResult(CloseableReference<T> result) {
    CloseableReference.closeSafely(result);
  }

  @Override
  protected void onNewResultImpl(CloseableReference<T> result, boolean isLast) {
    super.onNewResultImpl(CloseableReference.cloneOrNull(result), isLast);
  }
}

```

可以看到没有什么逻辑，应该都在父类`AbstractProducerToDataSourceAdapter`中：

```
/**
 * DataSource<T> backed by a Producer<T>
 *
 * @param <T>
 */
@ThreadSafe
public abstract class AbstractProducerToDataSourceAdapter<T> extends AbstractDataSource<T> {

  private final SettableProducerContext mSettableProducerContext;
  private final RequestListener mRequestListener;

  protected AbstractProducerToDataSourceAdapter(
      Producer<T> producer,
      SettableProducerContext settableProducerContext,
      RequestListener requestListener) {
    mSettableProducerContext = settableProducerContext;
    mRequestListener = requestListener;
    mRequestListener.onRequestStart(
        settableProducerContext.getImageRequest(),
        mSettableProducerContext.getCallerContext(),
        mSettableProducerContext.getId(),
        mSettableProducerContext.isPrefetch());
    producer.produceResults(createConsumer(), settableProducerContext);  //看这里这里这里！！！
  }

  private Consumer<T> createConsumer() {
    return new BaseConsumer<T>() {
      @Override
      protected void onNewResultImpl(@Nullable T newResult, boolean isLast) {
        AbstractProducerToDataSourceAdapter.this.onNewResultImpl(newResult, isLast);
      }

      @Override
      protected void onFailureImpl(Throwable throwable) {
        AbstractProducerToDataSourceAdapter.this.onFailureImpl(throwable);
      }

      @Override
      protected void onCancellationImpl() {
        AbstractProducerToDataSourceAdapter.this.onCancellationImpl();
      }

      @Override
      protected void onProgressUpdateImpl(float progress) {
        AbstractProducerToDataSourceAdapter.this.setProgress(progress);
      }
    };
  }

  protected void onNewResultImpl(@Nullable T result, boolean isLast) {
    if (super.setResult(result, isLast)) {
      if (isLast) {
        mRequestListener.onRequestSuccess(
            mSettableProducerContext.getImageRequest(),
            mSettableProducerContext.getId(),
            mSettableProducerContext.isPrefetch());
      }
    }
  }
......
}
```

看到了吧，第23行，在构造方法里，调用了`Producer`的`produceResults`,这样`Producer`的流程就串联起来了:

![58EE2FC6-8B6A-4A09-ADBA-2C17DE085D10](http://oon96myva.bkt.clouddn.com/md/hlqh9.png)

另一方面，最初的`Consumer`也是在这创建的，在`createConsumer`方法里创建了匿名内部类继承了`BaseConsumer`,`BaseConsumer`对`Consumer`的方法进行了异常处理，自定义的`Consumser`都推荐继承该类，上边看到的`DelegatingConsumer`也是继承于此。

我们看一下`Consumer`的回调实现:

```
 @Override
      protected void onNewResultImpl(@Nullable T newResult, boolean isLast) {
        AbstractProducerToDataSourceAdapter.this.onNewResultImpl(newResult, isLast);
      }
      
        protected void onNewResultImpl(@Nullable T result, boolean isLast) {
    if (super.setResult(result, isLast)) {
      if (isLast) {
        mRequestListener.onRequestSuccess(
            mSettableProducerContext.getImageRequest(),
            mSettableProducerContext.getId(),
            mSettableProducerContext.isPrefetch());
      }
    }
  }
```

具体实现在`super.setResult`里：

```
protected boolean setResult(@Nullable T value, boolean isLast) {
    boolean result = setResultInternal(value, isLast);
    if (result) {
      notifyDataSubscribers();
    }
    return result;
  }
private void notifyDataSubscribers() {
    final boolean isFailure = hasFailed();
    final boolean isCancellation = wasCancelled();
    for (Pair<DataSubscriber<T>, Executor> pair : mSubscribers) {
      notifyDataSubscriber(pair.first, pair.second, isFailure, isCancellation);
    }
  }

  private void notifyDataSubscriber(
      final DataSubscriber<T> dataSubscriber,
      final Executor executor,
      final boolean isFailure,
      final boolean isCancellation) {
    executor.execute(
        new Runnable() {
          @Override
          public void run() {
            if (isFailure) {
              dataSubscriber.onFailure(AbstractDataSource.this);
            } else if (isCancellation) {
              dataSubscriber.onCancellation(AbstractDataSource.this);
            } else {
              dataSubscriber.onNewResult(AbstractDataSource.this);
            }
          }
        });
  }
```

最终调用了`dataSubscriber`的回调，我们知道`DataSource`和`DataSubcriber`组成了观察者模式,`AbstractProducerToDataSourceAdapter`继承了`DataSource`,那`dataSubscriber`是从哪里来的呢?我们先往下看

## 2.mPipelineDraweeControllerFactory.newController

到这里，可能都已经忘了前边的逻辑，现在回顾一下：

![C8A165B9-E450-45C0-AC7E-2604398E8FFF](http://oon96myva.bkt.clouddn.com/md/27sfq.png)

现在到了第11步...

```
  public PipelineDraweeController newController(
      Supplier<DataSource<CloseableReference<CloseableImage>>> dataSourceSupplier,
      String id,
      Object callerContext) {
    return new PipelineDraweeController(
        mResources,
        mDeferredReleaser,
        mAnimatedDrawableFactory,
        mUiThreadExecutor,
        dataSourceSupplier,
        id,
        callerContext);
  }
```

这里只是对PipelineDraweeController实例化。

# 3.setController()

## 1.Fresco MVC

```
 /** Sets the controller. */
  public void setController(@Nullable DraweeController draweeController) {
    mDraweeHolder.setController(draweeController);
    super.setImageDrawable(mDraweeHolder.getTopLevelDrawable());
  }
```

这里有个`mDraweeHolder`,看一下这个类的定义：

```
/**
 * A holder class for Drawee controller and hierarchy.
 *
 * <p>Drawee users, should, as a rule, use {@link DraweeView} or its subclasses. There are
 * situations where custom views are required, however, and this class is for those circumstances.
 */
public class DraweeHolder<DH extends DraweeHierarchy> implements VisibilityCallback {

  private DH mHierarchy;
  private DraweeController mController = null;
  ...
}
```

`DraweeHolder`持有了`DraweeHierarchy`和`DraweeController`实例

**Fresco** 是一个典型的 MVC 模型，只不过把 **Model** 叫做 `DraweeHierarchy`。

- M : DraweeHierarchy
- V : DraweeView（DraweeHolder）
- C : DraweeController

1. View 对应`DraweeView`类（实际上是`DraweeHolder`），其负责展示数据，显示图片。 
2. Model对应`DraweeHierarchy`类，其负责持有数据，用一个层级组织和维护最终绘制和显示的图片。 
3. Controller对应`DraweeController`类，其负责控制数据的逻辑。

![D12B1518-FFC4-4000-9953-83508D35C1A3](http://oon96myva.bkt.clouddn.com/md/dx2ml.png)

从DraweeHolder的注释可以看出，这是一个解耦的设计，当我们想自己定义一个 `ImageView` 而不是使用 `DraweeView` 时，通过 `ViewHolder` 一样可以使用其他两个组件.

```
public class DraweeView<DH extends DraweeHierarchy> extends ImageView {
  // other methods and properties

  /** Sets the hierarchy. */
  public void setHierarchy(DH hierarchy) {
    mDraweeHolder.setHierarchy(hierarchy);
    super.setImageDrawable(mDraweeHolder.getTopLevelDrawable());
  }

  /** Sets the controller. */
  public void setController(@Nullable DraweeController draweeController) {
    mDraweeHolder.setController(draweeController);
    super.setImageDrawable(mDraweeHolder.getTopLevelDrawable());
  }
}
```

每次为 `DraweeView` 设置 `hierarchy` 或 `controller` 时，会同时通过 `super.setImageDrawable(mDraweeHolder.getTopLevelDrawable())` 更新需要显示的图像。

```
public interface DraweeHierarchy {

  /**
   * Returns the top level drawable in the corresponding hierarchy. Hierarchy should always have
   * the same instance of its top level drawable.
   * @return top level drawable
   */
  public Drawable getTopLevelDrawable();
}
```

`DraweeHierarchy` 只定义了一个方法 - `getTopLevelDrawable`

```
/**
 * Interface that represents a Drawee controller used by a DraweeView.
 * <p> The view forwards events to the controller. The controller controls
 * its hierarchy based on those events.
 */
public interface DraweeController {

  /** Gets the hierarchy. */
  @Nullable
  DraweeHierarchy getHierarchy();

  /** Sets a new hierarchy. */
  void setHierarchy(@Nullable DraweeHierarchy hierarchy);

  /**
   * Called when the view containing the hierarchy is attached to a window
   * (either temporarily or permanently).
   */
  void onAttach();

  /**
   * Called when the view containing the hierarchy is detached from a window
   * (either temporarily or permanently).
   */
  void onDetach();

  /**
   * Called when the view containing the hierarchy receives a touch event.
   * @return true if the event was handled by the controller, false otherwise
   */
  boolean onTouchEvent(MotionEvent event);

  /**
   * For an animated image, returns an Animatable that lets clients control the animation.
   * @return animatable, or null if the image is not animated or not loaded yet
   */
  Animatable getAnimatable();

}
```

`DraweeController`暴露了设置 `hierarchy` 和接收 `Event` 的方法。

```
void setHierarchy(@Nullable DraweeHierarchy hierarchy)
public boolean onTouchEvent(MotionEvent event)
```

## 2.mDraweeHolder.setController

```
 /**
   * Sets a new controller.
   */
  public void setController(@Nullable DraweeController draweeController) {
    boolean wasAttached = mIsControllerAttached;
    if (wasAttached) {
      detachController();
    }

    // Clear the old controller
    if (mController != null) {
      mEventTracker.recordEvent(Event.ON_CLEAR_OLD_CONTROLLER);
      mController.setHierarchy(null);
    }
    mController = draweeController;
    if (mController != null) {
      mEventTracker.recordEvent(Event.ON_SET_CONTROLLER);
      mController.setHierarchy(mHierarchy);
    } else {
      mEventTracker.recordEvent(Event.ON_CLEAR_CONTROLLER);
    }

    if (wasAttached) {
      attachController();
    }
  }

```

这里调用了`attachController`:

```
private void attachController() {
    if (mIsControllerAttached) {
      return;
    }
    mEventTracker.recordEvent(Event.ON_ATTACH_CONTROLLER);
    mIsControllerAttached = true;
    if (mController != null &&
        mController.getHierarchy() != null) {
      mController.onAttach();
    }
  }
```

调用了` mController.onAttach()`,实现在`PipelineDraweeController`的父类`AbstractDraweeController`中：

```
@Override
  public void onAttach() {
    ......
    if (!mIsRequestSubmitted) {
      submitRequest();
    }
  }
  
 protected void submitRequest() {
    mEventTracker.recordEvent(Event.ON_DATASOURCE_SUBMIT);
    getControllerListener().onSubmit(mId, mCallerContext);
    mSettableDraweeHierarchy.setProgress(0, true);
    mIsRequestSubmitted = true;
    mHasFetchFailed = false;
    mDataSource = getDataSource();
    if (FLog.isLoggable(FLog.VERBOSE)) {
      FLog.v(
          TAG,
          "controller %x %s: submitRequest: dataSource: %x",
          System.identityHashCode(this),
          mId,
          System.identityHashCode(mDataSource));
    }
    final String id = mId;
    final boolean wasImmediate = mDataSource.hasResult();
    final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            // isFinished must be obtained before image, otherwise we might set intermediate result
            // as final image.
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            T image = dataSource.getResult();
            if (image != null) {
              onNewResultInternal(id, dataSource, image, progress, isFinished, wasImmediate);
            } else if (isFinished) {
              onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
            }
          }
          @Override
          public void onFailureImpl(DataSource<T> dataSource) {
            onFailureInternal(id, dataSource, dataSource.getFailureCause(), /* isFinished */ true);
          }
          @Override
          public void onProgressUpdate(DataSource<T> dataSource) {
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            onProgressUpdateInternal(id, dataSource, progress, isFinished);
          }
        };
    mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
  }
```

找到了吧，这里就是我们上边提到的`DataSubscriber`,这个观察者模式就通了，看一下`  onNewResultInternal`:

```
 private void onNewResultInternal(
      String id,
      DataSource<T> dataSource,
      @Nullable T image,
      float progress,
      boolean isFinished,
      boolean wasImmediate) {
    ......
    try {
      // set the new image
      if (isFinished) {
        logMessageAndImage("set_final_result @ onNewResult", image);
        mDataSource = null;
        mSettableDraweeHierarchy.setImage(drawable, 1f, wasImmediate);
        getControllerListener().onFinalImageSet(id, getImageInfo(image), getAnimatable());
        // IMPORTANT: do not execute any instance-specific code after this point
      } else {
        logMessageAndImage("set_intermediate_result @ onNewResult", image);
        mSettableDraweeHierarchy.setImage(drawable, progress, wasImmediate);
        getControllerListener().onIntermediateImageSet(id, getImageInfo(image));
        // IMPORTANT: do not execute any instance-specific code after this point
      }
    } finally {
      if (previousDrawable != null && previousDrawable != drawable) {
        releaseDrawable(previousDrawable);
      }
      if (previousImage != null && previousImage != image) {
        logMessageAndImage("release_previous_result @ onNewResult", previousImage);
        releaseImage(previousImage);
      }
    }
  }
```

这里的`   mSettableDraweeHierarchy.setImage(drawable, 1f, wasImmediate);`就是设置显示的图片啦

具体实现在`DrawableHierarchy`的子类`GenericDraweeHierarchy`中,先看下这个类的定义:

![909141F1-9AAE-4AD2-B1D0-4C5DBA1CD870](http://oon96myva.bkt.clouddn.com/md/b89by.png)

 可以看出来，fresco显示图片的时候，用到了很多层级，占位图片，失败时的图片，重试时的图片，进度图片，所有的层级drawable都在`FadeDrawable`中

```
 private final int mPlaceholderImageIndex;
  private final int mProgressBarImageIndex;
  private final int mActualImageIndex;
  private final int mRetryImageIndex;
  private final int mFailureImageIndex;

  GenericDraweeHierarchy(GenericDraweeHierarchyBuilder builder) {
    mResources = builder.getResources();
    mRoundingParams = builder.getRoundingParams();

    mActualImageWrapper = new ForwardingDrawable(mEmptyActualImageDrawable);

    int numBackgrounds = (builder.getBackgrounds() != null) ? builder.getBackgrounds().size() : 0;
    int numOverlays = (builder.getOverlays() != null) ? builder.getOverlays().size() : 0;
    numOverlays += (builder.getPressedStateOverlay() != null) ? 1 : 0;

    // layer indices and count
    int numLayers = 0;
    int backgroundsIndex = numLayers;
    numLayers += numBackgrounds;
    mPlaceholderImageIndex = numLayers++;
    mActualImageIndex = numLayers++;
    mProgressBarImageIndex = numLayers++;
    mRetryImageIndex = numLayers++;
    mFailureImageIndex = numLayers++;
    int overlaysIndex = numLayers;
    numLayers += numOverlays;

    // array of layers
    Drawable[] layers = new Drawable[numLayers];
    if (numBackgrounds > 0) {
      int index = 0;
      for (Drawable background : builder.getBackgrounds()) {
        layers[backgroundsIndex + index++] = buildBranch(background, null);
      }
    }
    layers[mPlaceholderImageIndex] = buildBranch(
        builder.getPlaceholderImage(),
        builder.getPlaceholderImageScaleType());
    layers[mActualImageIndex] = buildActualImageBranch(
        mActualImageWrapper,
        builder.getActualImageScaleType(),
        builder.getActualImageFocusPoint(),
        builder.getActualImageMatrix(),
        builder.getActualImageColorFilter());
    layers[mProgressBarImageIndex] = buildBranch(
        builder.getProgressBarImage(),
        builder.getProgressBarImageScaleType());
    layers[mRetryImageIndex] = buildBranch(
        builder.getRetryImage(),
        builder.getRetryImageScaleType());
    layers[mFailureImageIndex] = buildBranch(
        builder.getFailureImage(),
        builder.getFailureImageScaleType());
    if (numOverlays > 0) {
      int index = 0;
      if (builder.getOverlays() != null) {
        for (Drawable overlay : builder.getOverlays()) {
          layers[overlaysIndex + index++] = buildBranch(overlay, null);
        }
      }
      if (builder.getPressedStateOverlay() != null) {
        layers[overlaysIndex + index] = buildBranch(builder.getPressedStateOverlay(), null);
      }
    }

    // fade drawable composed of layers
    mFadeDrawable = new FadeDrawable(layers);
    mFadeDrawable.setTransitionDuration(builder.getFadeDuration());

    // rounded corners drawable (optional)
    Drawable maybeRoundedDrawable =
        WrappingUtils.maybeWrapWithRoundedOverlayColor(mFadeDrawable, mRoundingParams);

    // top-level drawable
    mTopLevelDrawable = new RootDrawable(maybeRoundedDrawable);
    mTopLevelDrawable.mutate();

    resetFade();
  }
```

可以看到，在`GenericDraweeHierarchy`的构造方法中，各个层级的图片存放在了`layers`数组中，最后作为构造参数传给了`FadeDrawable`，那`GenericDraweeHierarchy`是什么时候创建的呢?

```
public GenericDraweeHierarchy build() {
    validate();
    return new GenericDraweeHierarchy(this);
  }
```

实现在`GenericDraweeHierarchyBuilder`中，这里用了Builder模式，这个类又是啥时候创建的呢？

```

  public GenericDraweeView(Context context, AttributeSet attrs, int defStyle) {
    super(context, attrs, defStyle);
    inflateHierarchy(context, attrs);
  }

  @TargetApi(Build.VERSION_CODES.LOLLIPOP)
  public GenericDraweeView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
    super(context, attrs, defStyleAttr, defStyleRes);
    inflateHierarchy(context, attrs);
  }

  private void inflateHierarchy(Context context, @Nullable AttributeSet attrs) {
    Resources resources = context.getResources();

    // fading animation defaults
    int fadeDuration = GenericDraweeHierarchyBuilder.DEFAULT_FADE_DURATION;
    // images & scale types defaults
    int placeholderId = 0;
    ScalingUtils.ScaleType placeholderScaleType
        = GenericDraweeHierarchyBuilder.DEFAULT_SCALE_TYPE;
    int retryImageId = 0;
    ScalingUtils.ScaleType retryImageScaleType =
        GenericDraweeHierarchyBuilder.DEFAULT_SCALE_TYPE;
    int failureImageId = 0;
    ScalingUtils.ScaleType failureImageScaleType =
        GenericDraweeHierarchyBuilder.DEFAULT_SCALE_TYPE;
    int progressBarId = 0;
    ScalingUtils.ScaleType progressBarScaleType =
        GenericDraweeHierarchyBuilder.DEFAULT_SCALE_TYPE;
    ScalingUtils.ScaleType actualImageScaleType =
        GenericDraweeHierarchyBuilder.DEFAULT_ACTUAL_IMAGE_SCALE_TYPE;
    int backgroundId = 0;
    int overlayId = 0;
    int pressedStateOverlayId = 0;
    // rounding defaults
    boolean roundAsCircle = false;
    int roundedCornerRadius = 0;
    boolean roundTopLeft = true;
    boolean roundTopRight = true;
    boolean roundBottomRight = true;
    boolean roundBottomLeft = true;
    int roundWithOverlayColor = 0;
    int roundingBorderWidth = 0;
    int roundingBorderColor = 0;
    int roundingBorderPadding = 0;
    int progressBarAutoRotateInterval = 0;


    if (attrs != null) {
      TypedArray gdhAttrs = context.obtainStyledAttributes(
          attrs,
          R.styleable.GenericDraweeView);

      try {
        final int indexCount = gdhAttrs.getIndexCount();

        for (int i = 0; i < indexCount; i++) {
          final int idx = gdhAttrs.getIndex(i);

          // most popular ones first
          if (idx == R.styleable.GenericDraweeView_actualImageScaleType) {
            // actual image scale type
            actualImageScaleType = getScaleTypeFromXml(
                gdhAttrs,
                R.styleable.GenericDraweeView_actualImageScaleType,
                actualImageScaleType);

          } else if (idx == R.styleable.GenericDraweeView_placeholderImage) {
            // placeholder image
            placeholderId = gdhAttrs.getResourceId(
                R.styleable.GenericDraweeView_placeholderImage,
                placeholderId);

          } else if (idx == R.styleable.GenericDraweeView_pressedStateOverlayImage) {
            // pressedState overlay
            pressedStateOverlayId = gdhAttrs.getResourceId(
                R.styleable.GenericDraweeView_pressedStateOverlayImage,
                pressedStateOverlayId);

          } else if (idx == R.styleable.GenericDraweeView_progressBarImage) {
            // progress bar image
            progressBarId = gdhAttrs.getResourceId(
                R.styleable.GenericDraweeView_progressBarImage,
                progressBarId);

          // the remaining ones without any particular order
          } else if (idx == R.styleable.GenericDraweeView_fadeDuration) {
            // fade duration
            fadeDuration = gdhAttrs.getInt(
                R.styleable.GenericDraweeView_fadeDuration,
                fadeDuration);

          } else if (idx == R.styleable.GenericDraweeView_viewAspectRatio) {
            // aspect ratio
            setAspectRatio(gdhAttrs.getFloat(
                R.styleable.GenericDraweeView_viewAspectRatio,
                getAspectRatio()));

          } else if (idx == R.styleable.GenericDraweeView_placeholderImageScaleType) {
            // placeholder image scale type
            placeholderScaleType = getScaleTypeFromXml(
                gdhAttrs,
                R.styleable.GenericDraweeView_placeholderImageScaleType,
                placeholderScaleType);

          } else if (idx == R.styleable.GenericDraweeView_retryImage) {
            // retry image
            retryImageId = gdhAttrs.getResourceId(
                R.styleable.GenericDraweeView_retryImage,
                retryImageId);

          } else if (idx == R.styleable.GenericDraweeView_retryImageScaleType) {
            // retry image scale type
            retryImageScaleType = getScaleTypeFromXml(
                gdhAttrs,
                R.styleable.GenericDraweeView_retryImageScaleType,
                retryImageScaleType);

          } else if (idx == R.styleable.GenericDraweeView_failureImage) {
            // failure image
            failureImageId = gdhAttrs.getResourceId(
                R.styleable.GenericDraweeView_failureImage,
                failureImageId);

          } else if (idx == R.styleable.GenericDraweeView_failureImageScaleType) {
            // failure image scale type
            failureImageScaleType = getScaleTypeFromXml(
                gdhAttrs,
                R.styleable.GenericDraweeView_failureImageScaleType,
                failureImageScaleType);

          } else if (idx == R.styleable.GenericDraweeView_progressBarImageScaleType) {
            // progress bar image scale type
            progressBarScaleType = getScaleTypeFromXml(
                gdhAttrs,
                R.styleable.GenericDraweeView_progressBarImageScaleType,
                progressBarScaleType);

          } else if (idx == R.styleable.GenericDraweeView_progressBarAutoRotateInterval) {
            // progress bar auto rotate interval
            progressBarAutoRotateInterval = gdhAttrs.getInteger(
                R.styleable.GenericDraweeView_progressBarAutoRotateInterval,
                0);

          } else if (idx == R.styleable.GenericDraweeView_backgroundImage) {
            // background
            backgroundId = gdhAttrs.getResourceId(
                R.styleable.GenericDraweeView_backgroundImage,
                backgroundId);

          } else if (idx == R.styleable.GenericDraweeView_overlayImage) {
            // overlay
            overlayId = gdhAttrs.getResourceId(
                R.styleable.GenericDraweeView_overlayImage,
                overlayId);

          } else if (idx == R.styleable.GenericDraweeView_roundAsCircle) {
            // rounding parameters
            roundAsCircle = gdhAttrs.getBoolean(
                R.styleable.GenericDraweeView_roundAsCircle,
                roundAsCircle);

          } else if (idx == R.styleable.GenericDraweeView_roundedCornerRadius) {
            roundedCornerRadius = gdhAttrs.getDimensionPixelSize(
                R.styleable.GenericDraweeView_roundedCornerRadius,
                roundedCornerRadius);

          } else if (idx == R.styleable.GenericDraweeView_roundTopLeft) {
            roundTopLeft = gdhAttrs.getBoolean(
                R.styleable.GenericDraweeView_roundTopLeft,
                roundTopLeft);

          } else if (idx == R.styleable.GenericDraweeView_roundTopRight) {
            roundTopRight = gdhAttrs.getBoolean(
                R.styleable.GenericDraweeView_roundTopRight,
                roundTopRight);

          } else if (idx == R.styleable.GenericDraweeView_roundBottomRight) {
            roundBottomRight = gdhAttrs.getBoolean(
                R.styleable.GenericDraweeView_roundBottomRight,
                roundBottomRight);

          } else if (idx == R.styleable.GenericDraweeView_roundBottomLeft) {
            roundBottomLeft = gdhAttrs.getBoolean(
                R.styleable.GenericDraweeView_roundBottomLeft,
                roundBottomLeft);

          } else if (idx == R.styleable.GenericDraweeView_roundWithOverlayColor) {
            roundWithOverlayColor = gdhAttrs.getColor(
                R.styleable.GenericDraweeView_roundWithOverlayColor,
                roundWithOverlayColor);

          } else if (idx == R.styleable.GenericDraweeView_roundingBorderWidth) {
            roundingBorderWidth = gdhAttrs.getDimensionPixelSize(
                R.styleable.GenericDraweeView_roundingBorderWidth,
                roundingBorderWidth);

          } else if (idx == R.styleable.GenericDraweeView_roundingBorderColor) {
            roundingBorderColor = gdhAttrs.getColor(
                R.styleable.GenericDraweeView_roundingBorderColor,
                roundingBorderColor);

          } else if (idx == R.styleable.GenericDraweeView_roundingBorderPadding) {
            roundingBorderPadding = gdhAttrs.getDimensionPixelSize(
                R.styleable.GenericDraweeView_roundingBorderPadding,
                roundingBorderPadding);

          }
        }
      } finally {
        gdhAttrs.recycle();
      }
    }

    GenericDraweeHierarchyBuilder builder = new GenericDraweeHierarchyBuilder(resources);
    // set fade duration
    builder.setFadeDuration(fadeDuration);
    // set images & scale types
    if (placeholderId > 0) {
      builder.setPlaceholderImage(resources.getDrawable(placeholderId), placeholderScaleType);
    }
    if (retryImageId > 0) {
      builder.setRetryImage(resources.getDrawable(retryImageId), retryImageScaleType);
    }
    if (failureImageId > 0) {
      builder.setFailureImage(resources.getDrawable(failureImageId), failureImageScaleType);
    }
    if (progressBarId > 0) {
      Drawable progressBarDrawable = resources.getDrawable(progressBarId);
      if (progressBarAutoRotateInterval > 0) {
        progressBarDrawable =
            new AutoRotateDrawable(progressBarDrawable, progressBarAutoRotateInterval);
      }
      builder.setProgressBarImage(progressBarDrawable, progressBarScaleType);
    }
    if (backgroundId > 0) {
      builder.setBackground(resources.getDrawable(backgroundId));
    }
    if (overlayId > 0) {
      builder.setOverlay(resources.getDrawable(overlayId));
    }
    if (pressedStateOverlayId > 0) {
      builder.setPressedStateOverlay(getResources().getDrawable(pressedStateOverlayId));
    }

    builder.setActualImageScaleType(actualImageScaleType);
    // set rounding parameters
    if (roundAsCircle || roundedCornerRadius > 0) {
      RoundingParams roundingParams = new RoundingParams();
      roundingParams.setRoundAsCircle(roundAsCircle);
      if (roundedCornerRadius > 0) {
        roundingParams.setCornersRadii(
            roundTopLeft ? roundedCornerRadius : 0,
            roundTopRight ? roundedCornerRadius : 0,
            roundBottomRight ? roundedCornerRadius : 0,
            roundBottomLeft ? roundedCornerRadius : 0);
      }
      if (roundWithOverlayColor != 0) {
        roundingParams.setOverlayColor(roundWithOverlayColor);
      }
      if (roundingBorderColor != 0 && roundingBorderWidth > 0) {
        roundingParams.setBorder(roundingBorderColor, roundingBorderWidth);
      }
      if (roundingBorderPadding != 0) {
        roundingParams.setPadding(roundingBorderPadding);
      }
      builder.setRoundingParams(roundingParams);
    }
    setHierarchy(builder.build());
  }

```

`GenericDraweeView`是`SimpleDraweeView`的父类，我们可以看到，这里负责解析xml，并且最后创建了Hierarchy.

我们重新看一下`setImage`方法：

```

  @Override
  public void setImage(Drawable drawable, float progress, boolean immediate) {
    drawable = WrappingUtils.maybeApplyLeafRounding(drawable, mRoundingParams, mResources);
    drawable.mutate();
    mActualImageWrapper.setDrawable(drawable);  //真正显示图片(ForwardingDrawable)
    mFadeDrawable.beginBatchMode();
    fadeOutBranches();  //渐变图层
    fadeInLayer(mActualImageIndex);
    setProgress(progress);  
    if (immediate) {
      mFadeDrawable.finishTransitionImmediately();  //刷新
    }
    mFadeDrawable.endBatchMode();
  }
```

`fadeInLayer(mActualImageIndex)`:

```
private void fadeInLayer(int index) {
    if (index >= 0) {
      mFadeDrawable.fadeInLayer(index);
    }
  }
```

调用了`FadeDrawable`的`fadeInlayer`控制图层显示:

```
 /**
   * Starts fading in the specified layer.
   * @param index the index of the layer to fade in.
   */
  public void fadeInLayer(int index) {
    mTransitionState = TRANSITION_STARTING;
    mIsLayerOn[index] = true;
    invalidateSelf();
  }
```

这里更改了`mTransitionState`,` mIsLayerOn[index] = true;`,调用了`invalidateSelf`,肯定会走`draw`方法:

```
 @Override
  public void draw(Canvas canvas) {
    boolean done = true;
    float ratio;

    switch (mTransitionState) {
      case TRANSITION_STARTING:
        // initialize start alphas and start time
        System.arraycopy(mAlphas, 0, mStartAlphas, 0, mLayers.length);
        mStartTimeMs = getCurrentTimeMs();
        // if the duration is 0, update alphas to the target opacities immediately
        ratio = (mDurationMs == 0) ? 1.0f : 0.0f;
        // if all the layers have reached their target opacity, transition is done
        done = updateAlphas(ratio);  //更新了alpha
        mTransitionState = done ? TRANSITION_NONE : TRANSITION_RUNNING;
        break;

      case TRANSITION_RUNNING:
        Preconditions.checkState(mDurationMs > 0);
        // determine ratio based on the elapsed time
        ratio = (float) (getCurrentTimeMs() - mStartTimeMs) / mDurationMs;
        // if all the layers have reached their target opacity, transition is done
        done = updateAlphas(ratio);
        mTransitionState = done ? TRANSITION_NONE : TRANSITION_RUNNING;
        break;

      case TRANSITION_NONE:
        // there is no transition in progress and mAlphas should be left as is.
        done = true;
        break;
    }

    for (int i = 0; i < mLayers.length; i++) {  //所有的层画在同一个canvas上
      drawDrawableWithAlpha(canvas, mLayers[i], mAlphas[i] * mAlpha / 255);
    }

    if (!done) {
      invalidateSelf();
    }
  }
```

通过`updateAlphas`更新某一layer的alpha :

```
  /**
   * Updates the current alphas based on the ratio of the elapsed time and duration.
   * @param ratio
   * @return whether the all layers have reached their target opacity
   */
  private boolean updateAlphas(float ratio) {
    boolean done = true;
    for (int i = 0; i < mLayers.length; i++) {
      int dir = mIsLayerOn[i] ? +1 : -1;
      // determines alpha value and clamps it to [0, 255]
      mAlphas[i] = (int) (mStartAlphas[i] + dir * 255 * ratio);
      if (mAlphas[i] < 0) {
        mAlphas[i] = 0;
      }
      if (mAlphas[i] > 255) {
        mAlphas[i] = 255;
      }
      // determines whether the layer has reached its target opacity
      if (mIsLayerOn[i] && mAlphas[i] < 255) {
        done = false;
      }
      if (!mIsLayerOn[i] && mAlphas[i] > 0) {
        done = false;
      }
    }
    return done;
  }
```

因为上边设置了`mIsLayerOn[index] = true;`,这里的` mAlphas[i] = 255;`,

接着执行`draw`方法里de `drawDrawableWithAlpha`,所有的drawable都绘制了一起，只是通过alpha值来控制显示哪一drawable,这样最后图片就显示出来啦。写的很乱，谅解。



参考：https://blog.csdn.net/lufqnuli/article/details/51645556

https://blog.csdn.net/biezhihua/article/details/49862073

https://blog.csdn.net/ieyudeyinji/article/details/48260885

https://blog.csdn.net/feelang/article/details/45126421