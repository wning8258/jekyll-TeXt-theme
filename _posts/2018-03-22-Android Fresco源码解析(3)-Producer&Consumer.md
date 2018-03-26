---
layout: post
title: Android Fresco源码解析(3)-Producer&Consumer
key: 20180322 Android Fresco源码解析(3)-Producer&Consumer
tags: Android Fresco
typora-copy-images-to: ipic
---

参考地址：https://blog.csdn.net/biezhihua/article/details/49862073

`Producer`模式，在Fresco中请求产生这块，用到了大量的这种概念，将网络数据的获取，磁盘缓存获取，内存缓存获取，解码，编码和图片的变换等等的处理，分为模块处理，并以倒叙的方式联合再一起调用。

<!--more-->

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

`Produer`用于执行图片请求的各种任务，例如网络获取，硬盘缓存，内存缓存，解码，转换。通过`produceResults`把结果传递给`Consumer`,`Producer`支持链式操作。

为了成为一个链式，会有如下的代码模式：

```
public class XXProducer implements Producer {
  private final Producer mInputProducer;

  public XXProducer (Producer inputProducer) {
    mInputProducer = inputProducer;
  }

  @Override
  public void produceResults(Consumer consumer, ProducerContext context) {
    mInputProducer.produceResults(consumer, context);
  }
}
```

**接收一个外部的生产者，并在自身处理结果方法的最后调用外部生产者处理结果的方法。**

这样看似先创建外部的生产者，但是实际上最后才调用外部的生产者。所以Fresco实例的创建和反向调用就好像这个样子：

![363C66E4-88A0-49B6-B3CA-DC3C2E5E0B12](http://oon96myva.bkt.clouddn.com/md/4pzns.png)

`Producer`会通过观察者模式把变化通知给`Consumer`:

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

这里，同之前的`DataSource`一样，Fresco也给我们提供了一个非常好的抽象类，处理异常情况：

```

/**
 * Base implementation of Consumer that implements error handling conforming to the
 * Consumer's contract.
 *
 * <p> This class also prevents execution of callbacks if one of final methods was called before:
 * onFinish(isLast = true), onFailure or onCancellation.
 *
 * <p> All callbacks are executed within a synchronized block, so that clients can act as if all
 * callbacks are called on single thread.
 *
 * @param <T>
 */
@ThreadSafe
public abstract class BaseConsumer<T> implements Consumer<T> {

  /**
   * Set to true when onNewResult(isLast = true), onFailure or onCancellation is called. Further
   * calls to any of the 3 methods are not propagated
   */
  private boolean mIsFinished;

  public BaseConsumer() {
    mIsFinished = false;
  }

  @Override
  public synchronized void onNewResult(@Nullable T newResult, boolean isLast) {
    if (mIsFinished) {
      return;
    }
    mIsFinished = isLast;
    try {
      onNewResultImpl(newResult, isLast);
    } catch (Exception e) {
      onUnhandledException(e);
    }
  }

  @Override
  public synchronized void onFailure(Throwable t) {
    if (mIsFinished) {
      return;
    }
    mIsFinished = true;
    try {
      onFailureImpl(t);
    } catch (Exception e) {
      onUnhandledException(e);
    }
  }

  @Override
  public synchronized void onCancellation() {
    if (mIsFinished) {
      return;
    }
    mIsFinished = true;
    try {
      onCancellationImpl();
    } catch (Exception e) {
      onUnhandledException(e);
    }
  }

  /**
   * Called when the progress updates.
   *
   * @param progress in range [0, 1]
   */
  @Override
  public synchronized void onProgressUpdate(float progress) {
    if (mIsFinished) {
      return;
    }
    try {
      onProgressUpdateImpl(progress);
    } catch (Exception e) {
      onUnhandledException(e);
    }
  }

  /**
   * Called by onNewResult, override this method instead.
   */
  protected abstract void onNewResultImpl(T newResult, boolean isLast);

  /**
   * Called by onFailure, override this method instead
   */
  protected abstract void onFailureImpl(Throwable t);

  /**
   * Called by onCancellation, override this method instead
   */
  protected abstract void onCancellationImpl();

  /**
   * Called when the progress updates
   */
  protected void onProgressUpdateImpl(float progress) {
  }

  /**
   * Called whenever onNewResultImpl or onFailureImpl throw an exception
   */
  protected void onUnhandledException(Exception e) {
    FLog.wtf(this.getClass(), "unhandled exception", e);
  }
}

```

暴露出几个抽象方法去实现：

- onNewResultImpl
- onFailureImpl
- onCancellationImpl
- onProgressUpdateImpl

这里看一下Fresco中的实例应用：

```
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
    producer.produceResults(createConsumer(), settableProducerContext);
  }

  private Consumer<T> createConsumer() {  //创建Consumer
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

  private void onFailureImpl(Throwable throwable) {
    if (super.setFailure(throwable)) {
      mRequestListener.onRequestFailure(
          mSettableProducerContext.getImageRequest(),
          mSettableProducerContext.getId(),
          throwable,
          mSettableProducerContext.isPrefetch());
    }
  }

  private synchronized void onCancellationImpl() {
    Preconditions.checkState(isClosed());
  }
  。。。。。。

}
```

- `AbstractProducerToDataSourceAdapter`是`Producer`被调用`produceResults`的起始点，并且在第17行调用了`createConsumer`方法创建`Consumer`
- `createConsumer`里创建了一个`BaseConsumer`的一个匿名内部类，实现了上边提到的几个回调
- 回调方法里，利用`DataSource`和`DataSubscriber`通知给了DraweeController从而通知ui更新。