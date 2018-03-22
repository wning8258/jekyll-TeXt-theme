---
layout: post
title: Android Fresco源码解析(2)-图片显示
key: 20180322
tags: Android Fresco
typora-copy-images-to: ipic
---

Freso初始化的时候调用以下代码：

```
 /** Initializes Fresco with the default config. */
  public static void initialize(Context context) {
    ImagePipelineFactory.initialize(context);
    initializeDrawee(context);
  }
```

<!--more-->

`ImagePipelineFactory`是`ImagePipeline`的工厂类，那`ImagePipline`是什么？官方文档是这么说的：
`ImagePipeline` 负责完成加载图像，变成Android设备可呈现的形式所要做的每个事情。

大致流程如下:

- 检查内存缓存，如有，返回
- 后台线程开始后续工作
- 检查是否在未解码内存缓存中。如有，解码，变换，返回，然后缓存到内存缓存中。
- 检查是否在文件缓存中，如果有，变换，返回。缓存到未解码缓存和内存缓存中。
- 从网络或者本地加载。加载完成后，解码，变换，返回。存到各个缓存中。

既然本身就是一个图片加载组件，那么一图胜千言。

![453B5E84-FEDD-4770-BDA9-1F775FC8877A](http://oon96myva.bkt.clouddn.com/md/6yguy.png)

`ImagePipeline` 可以从本地文件加载文件，也可以从网络。支持PNG，GIF，WebP, JPEG.

### 1.实例化ImagePipelineFactory

`ImagePipelineFactory.initialize(context)`：

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

这里可以看到，`ImagePipelineFactory`的构造方法需要`ImagePipelineConfig`实例，如果不传的话，默认会用`ImagePipelineConfig.newBuilder(context).build()`创建实例，这里使用了Builder模式。

```
public class ImagePipelineConfig {
   private ImagePipelineConfig(Builder builder) {
       mAnimatedImageFactory = builder.mAnimatedImageFactory;
       mBitmapConfig =
        builder.mBitmapConfig == null ?
            Bitmap.Config.ARGB_8888 :
            builder.mBitmapConfig;
       ......
   }
   ......
   public static class Builder {

    private AnimatedImageFactory mAnimatedImageFactory;
    private Bitmap.Config mBitmapConfig;
    private Supplier<MemoryCacheParams> mBitmapMemoryCacheParamsSupplier;
    private CacheKeyFactory mCacheKeyFactory;
    private final Context mContext;
    private boolean mDownsampleEnabled = false;
    private boolean mWebpSupportEnabled = false;
    private boolean mDecodeFileDescriptorEnabled = mDownsampleEnabled;
    private boolean mDecodeMemoryFileEnabled;
    private Supplier<MemoryCacheParams> mEncodedMemoryCacheParamsSupplier;
    private ExecutorSupplier mExecutorSupplier;
    private ImageCacheStatsTracker mImageCacheStatsTracker;
    private ImageDecoder mImageDecoder;
    private Supplier<Boolean> mIsPrefetchEnabledSupplier;
    private DiskCacheConfig mMainDiskCacheConfig;
    private MemoryTrimmableRegistry mMemoryTrimmableRegistry;
    private NetworkFetcher mNetworkFetcher;
    private PlatformBitmapFactory mPlatformBitmapFactory;
    private PoolFactory mPoolFactory;
    private ProgressiveJpegConfig mProgressiveJpegConfig;
    private Set<RequestListener> mRequestListeners;
    private boolean mResizeAndRotateEnabledForNetwork = true;
    private DiskCacheConfig mSmallImageDiskCacheConfig;

    private Builder(Context context) {
      // Doesn't use a setter as always required.
      mContext = Preconditions.checkNotNull(context);
    }
    ......
     public ImagePipelineConfig build() {
      return new ImagePipelineConfig(this);
    }
    }
}
```

`build()`会创建一个 `ImagePipelineConfig` ，然后把 `this` 作为参数传给构造函数，而`ImagePipelineConfig` 的构造函数就是根据 `Builder` 来初始化。

初始化的策略非常简单：

- 如果 `builder` 中的参数值为空，则使用默认值。
- 如果 `builder` 中的参数值不为空，则使用 `Builder` 提供的值。

比如上边代码的第五行：

```
  mBitmapConfig =
        builder.mBitmapConfig == null ?
            Bitmap.Config.ARGB_8888 :
            builder.mBitmapConfig;
```

如果`builder.mBitmapConfig`为`null`,默认使用`Bitmap.Config.ARGB_8888`,否则使用`builder.mBitmapConfig`的值。

### 2.初始化Drawee

`initializeDrawee(context)`：

```
 private static void initializeDrawee(Context context) {
    sDraweeControllerBuilderSupplier = new PipelineDraweeControllerBuilderSupplier(context);
    SimpleDraweeView.initialize(sDraweeControllerBuilderSupplier);
  }
```

#### 2-1.实例化PipelineDraweeControllerBuilderSupplier

首先`new`出`PipelineDraweeControllerBuilderSupplier`实例，`PipelineDraweeControllerBuilderSupplier` 实现了`Supplier`接口，Supplier引用于Google Guava项目([https://github.com/google/guava](https://github.com/google/guava))

```
/**
 * A class that can supply objects of a single type.  Semantically, this could
 * be a factory, generator, builder, closure, or something else entirely. No
 * guarantees are implied by this interface.
 *
 * @author Harry Heymann
 * @since 2.0 (imported from Google Collections Library)
 */
public interface Supplier<T> {
  /**
   * Retrieves an instance of the appropriate type. The returned object may or
   * may not be a new instance, depending on the implementation.
   *
   * @return an instance of the appropriate type
   */
  T get();
}
```

`Supplier` 是一个**提供者**，用户包括但不限于`factory`, `generator`, `builder`, `closure`，接口方法 `get()` 用于返回它所提供的实例。

```
public class PipelineDraweeControllerBuilderSupplier implements
    Supplier<PipelineDraweeControllerBuilder> {

  private final Context mContext;
  private final ImagePipeline mImagePipeline;
  private final PipelineDraweeControllerFactory mPipelineDraweeControllerFactory;
  private final Set<ControllerListener> mBoundControllerListeners;

  public PipelineDraweeControllerBuilderSupplier(Context context) {
    this(context, ImagePipelineFactory.getInstance());
  }

  public PipelineDraweeControllerBuilderSupplier(
      Context context,
      ImagePipelineFactory imagePipelineFactory) {
    this(context, imagePipelineFactory, null);
  }

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
        UiThreadImmediateExecutorService.getInstance());
    mBoundControllerListeners = boundControllerListeners;
  }

  @Override
  public PipelineDraweeControllerBuilder get() {
    return new PipelineDraweeControllerBuilder(
        mContext,
        mPipelineDraweeControllerFactory,
        mImagePipeline,
        mBoundControllerListeners);
  }
}

```

`PipelineDraweeControllerBuilderSupplier`像是`PipelineDraweeControllerBuilder`的工厂类,通过`get`方法返回`PipelineDraweeControllerBuilder`实例。

- 从第10行看出，`PipelineDraweeControllerBuilderSupplier`使用了`ImagePipelineFactory.initialize(context);`创建出的`ImagePipelineFactory`的实例来实例化自己。
- 19行的构造方法，通过`ImagePipelineFactory.getImagePipeline`创建了ImagePipeline(上边简单讲过它的作用)；另外还创建了`PipelineDraweeControllerFactor`,它也是一个工厂类，用于创建`PipelineDraweeController`，DraweeController用于处理图片加载的逻辑，后面再说

#### 2-2 .初始化SimpleDraweeView

```
 SimpleDraweeView.initialize(sDraweeControllerBuilderSupplier);
```

`SimpleDraweeView`用于显示view,现在把上一步实例化的`sDraweeControllerBuilderSupplier`传入:

```
  /** Initializes {@link SimpleDraweeView} with supplier of Drawee controller builders. */
  public static void initialize(
      Supplier<? extends SimpleDraweeControllerBuilder> draweeControllerBuilderSupplier) {
    sDraweeControllerBuilderSupplier = draweeControllerBuilderSupplier;
  }
```

到此初始化的工作就完成了。
