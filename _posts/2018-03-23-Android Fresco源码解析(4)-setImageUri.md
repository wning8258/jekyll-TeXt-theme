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

## 1.PipelineDraweeControllerBuilder

PipelineDraweeControllerBuilder的继承关系如下：

![2684B59F-72D9-4560-B602-9679B76D9C0A](http://oon96myva.bkt.clouddn.com/md/nkqt4.png)

### 1.初始化

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

### 2. build()

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



#### 1.obtainDataSourceSupplier

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

这里的DataSource跟java的Future类似，具体可查看上一篇文章：
http://wning8258.com/2018/03/22/Android-Fresco%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90(2)-DataSource.html

这里对多个图片请求，多种质量请求都进行了处理，我们最关心的是`getDataSourceSupplierForRequest`方法，

简单来说，其实就是获得一个订阅者（DataSource），这个DataSource用来返回图片请求的数据和状态。

##### 1.getDataSourceSupplierForRequest

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

###### 1.getDecodedImageProducerSequence

这里又出现的新的概念，`Producer`

```
/**
 * Building block for image processing in the image pipeline.
 *
 * <p> Execution of image request consists of multiple different tasks such as network fetch,
 * disk caching, memory caching, decoding, applying transformations etc. Producer<T> represents
 * single task whose result is an instance of T. Breaking entire request into sequence of
 * Producers allows us to construct different requests while reusing the same blocks.
 *
 * <p> Producer supports multiple values and streaming.
 *
 * @param <T>
 */
public interface Producer<T> {

  /**
   * Start producing results for given context. Provided consumer is notified whenever progress is
   * made (new value is ready or error occurs).
   * @param consumer
   * @param context
   */
  void produceResults(Consumer<T> consumer, ProducerContext context);
}

```

`Produer`用于执行图片请求的各种任务，例如网络获取，硬盘缓存，内存缓存，解码，转换。通过`produceResults`把结果传递给`Consumer`

```
public interface Consumer<T> {

  /**
   * Called by a producer whenever new data is produced. This method should not throw an exception.
   *
   * <p> In case when result is closeable resource producer will close it after onNewResult returns.
   * Consumer needs to make copy of it if the resource must be accessed after that. Fortunately,
   * with CloseableReferences, that should not impose too much overhead.
   *
   * @param newResult
   * @param isLast true if newResult is the last result
   */
  void onNewResult(T newResult, boolean isLast);

  /**
   * Called by a producer whenever it terminates further work due to Throwable being thrown. This
   * method should not throw an exception.
   *
   * @param t
   */
  void onFailure(Throwable t);

  /**
   * Called by a producer whenever it is cancelled and won't produce any more results
   */
  void onCancellation();

  /**
   * Called when the progress updates.
   *
   * @param progress in range [0, 1]
   */
  void onProgressUpdate(float progress);
}

```

`Consumer`接收4种回调:

- onNewResult
- onFailure
- onCancellation
- onProgressUpdate

我们这里先看一下` mProducerSequenceFactory.getDecodedImageProducerSequence`做了什么。

`mProducerSequenceFactory`是个什么？在哪创建的?

![](http://oon96myva.bkt.clouddn.com/md/cz7qa.png)

可以看到，在Fresco初始化的时候，`mProducerSequenceFactory`就被创建了

看一它的`getDecodedImageProducerSequence`方法：

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
          newBitmapCacheGetToDecodeSequence(getCommonNetworkFetchToEncodedMemorySequence());
    }
    return mNetworkFetchSequence;
  }
```



###### 2.submitFetchRequest



#### 2.mPipelineDraweeControllerFactory.newController
