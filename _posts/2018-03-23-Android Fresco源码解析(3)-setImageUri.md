---
layout: post
title: Android Fresco源码解析(3)-setImageUri
key: 20180323  Android Fresco源码解析(3)-setImageUri
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

#### obtainDataSourceSupplier

实现在父类`AbstractDraweeControllerBuilder`中：

```

  /** Gets the top-level data source supplier to be used by a controller. */
  protected Supplier<DataSource<IMAGE>> obtainDataSourceSupplier() {
    if (mDataSourceSupplier != null) {
      return mDataSourceSupplier;
    }

    Supplier<DataSource<IMAGE>> supplier = null;

    // final image supplier;
    if (mImageRequest != null) {
      supplier = getDataSourceSupplierForRequest(mImageRequest);
    } else if (mMultiImageRequests != null) {
      supplier = getFirstAvailableDataSourceSupplier(mMultiImageRequests, mTryCacheOnlyFirst);
    }

    // increasing-quality supplier; highest-quality supplier goes first
    if (supplier != null && mLowResImageRequest != null) {
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

这里的DataSource跟java的Future类似，具体可查看：

[DataSource]: http://wning8258.com/2018/03/22/Android-Fresco%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90(2)-DataSource.html