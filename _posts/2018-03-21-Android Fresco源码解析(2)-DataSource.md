---
layout: post
title: Android Fresco源码解析(2)-DataSource
key: 20180321 Android Fresco源码解析(2)-DataSource
tags: Android Fresco
typora-copy-images-to: ipic
---
`DataSource`是Java中`Future`的替代品，`Future`只能返回一个结果，不同的是，`DataSource`可以产生一系列的结果；比如Fresco在完全加载完图片之前，会返回一系列渐进式的图像。`DataSource`必须调用close()方法以防止内存泄露。
<!--more-->

### 1.DataSource

```
 */
public interface DataSource<T> {

  /**
   * @return true if the data source is closed, false otherwise
   */
  boolean isClosed();

  /**
   * The most recent result of the asynchronous computation.
   *
   * <p>The caller gains ownership of the object and is responsible for releasing it.
   * Note that subsequent calls to getResult might give different results. Later results should be
   * considered to be of higher quality.
   *
   * <p>This method will return null in the following cases:
   * <ul>
   * <li>when the DataSource does not have a result ({@code hasResult} returns false).
   * <li>when the last result produced was null.
   * </ul>
   * @return current best result
   */
  @Nullable T getResult();

  /**
   * @return true if any result (possibly of lower quality) is available right now, false otherwise
   */
  boolean hasResult();

  /**
   * @return true if request is finished, false otherwise
   */
  boolean isFinished();

  /**
   * @return true if request finished due to error
   */
  boolean hasFailed();

  /**
   * @return failure cause if the source has failed, else null
   */
  @Nullable Throwable getFailureCause();

  /**
   * @return progress in range [0, 1]
   */
  float getProgress();

  /**
   * Cancels the ongoing request and releases all associated resources.
   *
   * <p>Subsequent calls to {@link #getResult} will return null.
   * @return true if the data source is closed for the first time
   */
  boolean close();

  /**
   * Subscribe for notifications whenever the state of the DataSource changes.
   *
   * <p>All changes will be observed on the provided executor.
   * @param dataSubscriber
   * @param executor
   */
  void subscribe(DataSubscriber<T> dataSubscriber, Executor executor);
}

```



DataSubscriber 和 DataSource 构成了一个观察者模式。

Datasource 提供了注册方法。

void subscribe(DataSubscriber dataSubscriber, Executor executor);

通过 subscribe 方法我们可以把 DataSubscriber 注册成为 DataSource 的观察者，然后当 DataSource 的数据发生变化时，在 Executor 中通知所有的观察者 - DataSubscriber。

DataSubscriber 会响应数据的四种变化。

- onNewResult
- onFailure
- onCancellation
- onProgressUpdate

使用Executor来通知观察者是比较高明的，这样做可以让回调方法的执行线程交由 DataSubscriber 来处理，增加了灵活性。

AbstractDataSource是DataSource的抽象实现，它管理了事件的各种状态，并且当状态变化时，会发送通知。
官方也建议其他的DataSource都继承AbstractDataSource

```

/**
 * An abstract implementation of {@link DataSource} interface.
 *
 * <p> It is highly recommended that other data sources extend this class as it takes care of the
 * state, as well as of notifying listeners when the state changes.
 *
 * <p> Subclasses should override {@link #closeResult(T result)} if results need clean up
 *
 * @param <T>
 */
public abstract class AbstractDataSource<T> implements DataSource<T> {
  /**
   * Describes state of data source
   */
  private enum DataSourceStatus {
    // data source has not finished yet
    IN_PROGRESS,

    // data source has finished with success
    SUCCESS,

    // data source has finished with failure
    FAILURE,
  }
   @GuardedBy("this")
  private DataSourceStatus mDataSourceStatus;
  private final ConcurrentLinkedQueue<Pair<DataSubscriber<T>, Executor>> mSubscribers;
  }
```

这里可以看出，`DataSource`有IN_PROGRESS,SUCCESS,FAILURE三种状态，当前的状态保存在变量`mDataSourceStatus`中，`mSubscribers`用于存放观察者。

```
@Override
  public void subscribe(final DataSubscriber<T> dataSubscriber, final Executor executor) {
    Preconditions.checkNotNull(dataSubscriber);
    Preconditions.checkNotNull(executor);
    boolean shouldNotify;

    synchronized(this) {
      if (mIsClosed) {
        return;
      }

      if (mDataSourceStatus == DataSourceStatus.IN_PROGRESS) {
        mSubscribers.add(Pair.create(dataSubscriber, executor));
      }

      shouldNotify = hasResult() || isFinished() || wasCancelled();
    }

    if (shouldNotify) {
      notifyDataSubscriber(dataSubscriber, executor, hasFailed(), wasCancelled()); //通知观察者
    }
  }
```

当状态发生改变时，调用`notifyDataSubscriber`通知观察者：

```

  protected boolean setResult(@Nullable T value, boolean isLast) {  //成功
    boolean result = setResultInternal(value, isLast);
    if (result) {
      notifyDataSubscribers();
    }
    return result;
  }

  protected boolean setFailure(Throwable throwable) { //失败
    boolean result = setFailureInternal(throwable);
    if (result) {
      notifyDataSubscribers();
    }
    return result;
  }

  protected boolean setProgress(float progress) {  //进度改变
    boolean result = setProgressInternal(progress);
    if (result) {
      notifyProgressUpdate();
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
 protected void notifyProgressUpdate() {
    for (Pair<DataSubscriber<T>, Executor> pair : mSubscribers) {
      final DataSubscriber<T> subscriber = pair.first;
      Executor executor = pair.second;
      executor.execute(
          new Runnable() {
            @Override
            public void run() {
              subscriber.onProgressUpdate(AbstractDataSource.this);  //进度更新
            }
          });
    }
  }
```

### 2.DataSubscriber

从上边可以看出，`DataSubscriber`接收四种状态的回调：

- onNewResult
- onProgressUpdate
- onFailure
- onCancellation

```
public interface DataSubscriber<T> {

  /**
   * Called whenever a new value is ready to be retrieved from the DataSource.
   *
   * <p>To retrieve the new value, call {@code dataSource.getResult()}.
   *
   * <p>To determine if the new value is the last, use {@code dataSource.isFinished()}.
   *
   * @param dataSource
   */
  void onNewResult(DataSource<T> dataSource);

  /**
   * Called whenever an error occurs inside of the pipeline.
   *
   * <p>No further results will be produced after this method is called.
   *
   * <p>The throwable resulting from the failure can be obtained using
   * {@code dataSource.getFailureCause}.
   *
   * @param dataSource
   */
  void onFailure(DataSource<T> dataSource);

  /**
   * Called whenever the request is cancelled (a request being cancelled means that is was closed
   * before it finished).
   *
   * <p>No further results will be produced after this method is called.
   *
   * @param dataSource
   */
  void onCancellation(DataSource<T> dataSource);

  /**
   * Called when the progress updates.
   *
   * @param dataSource
   */
  void onProgressUpdate(DataSource<T> dataSource);
}
```

回调结果都有一个DataSource参数，所有必须要调用close防止内存泄漏

这里，Fresco给我们提供了一个非常好的抽象类，防止忘记close...

```
public abstract class BaseDataSubscriber<T> implements DataSubscriber<T> {

  @Override
  public void onNewResult(DataSource<T> dataSource) {
    // isFinished() should be checked before calling onNewResultImpl(), otherwise
    // there would be a race condition: the final data source result might be ready before
    // we call isFinished() here, which would lead to the loss of the final result
    // (because of an early dataSource.close() call).
    final boolean shouldClose = dataSource.isFinished();
    try {
      onNewResultImpl(dataSource);
    } finally {
      if (shouldClose) {
        dataSource.close();
      }
    }
  }

  @Override
  public void onFailure(DataSource<T> dataSource) {
    try {
      onFailureImpl(dataSource);
    } finally {
      dataSource.close();
    }
  }

  @Override
  public void onCancellation(DataSource<T> dataSource) {
  }

  @Override
  public void onProgressUpdate(DataSource<T> dataSource) {
  }

  protected abstract void onNewResultImpl(DataSource<T> dataSource);

  protected abstract void onFailureImpl(DataSource<T> dataSource);
}

```

可以看到，在`onNewResult`和`onFailure`方法里增加了try catch，并且暴露新的抽象方法，`onNewResultImpl`和

`onFailureImpl`,大写的赞~\(≧▽≦)/~！！！



参考：https://blog.csdn.net/feelang/article/details/45420999
