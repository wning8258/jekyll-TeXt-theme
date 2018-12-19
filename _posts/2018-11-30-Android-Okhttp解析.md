---
layout: post
title: Android Okhttp解析
key: 20181130 Okhttp解析
tags: Android Okhttp
---

# OkHttp

## 1.OKhttp介绍

HTTP is the way modern applications network. It’s how we exchange data & media. Doing HTTP efficiently makes your stuff load faster and saves bandwidth.

OkHttp is an HTTP client that’s efficient by default:

- HTTP/2 support allows all requests to the same host to share a socket.
- Connection pooling reduces request latency (if HTTP/2 isn’t available).
- Transparent GZIP shrinks download sizes.
- Response caching avoids the network completely for repeat requests.

OkHttp perseveres when the network is troublesome: it will silently recover from common connection problems. If your service has multiple IP addresses OkHttp will attempt alternate addresses if the first connect fails. This is necessary for IPv4+IPv6 and for services hosted in redundant data centers. OkHttp initiates new connections with modern TLS features (SNI, ALPN), and falls back to TLS 1.0 if the handshake fails.

Using OkHttp is easy. Its request/response API is designed with fluent builders and immutability. It supports both synchronous blocking calls and async calls with callbacks.

OkHttp supports Android 2.3 and above. For Java, the minimum requirement is 1.7.

<!--more-->

OKHttp是一款高效的HTTP客户端，支持连接同一地址的链接共享同一个socket，通过连接池来减小响应延迟，

还有透明的GZIP压缩，请求缓存等优势，其核心主要有路由、连接协议、拦截器、代理、安全性认证、连接池以

及网络适配

## 2.原理

### 2.1基本使用

```
OkHttpClient mClient = new OkHttpClient();

public void void run(String url) throws IOException {
   Request request = new Request.Builder().url(finalUrl).build();
                mClient.newCall(request).enqueue(new Callback() {
                    @Override
                    public void onFailure(Call call, IOException e) {
                        LogUtils.i(TAG, "onFailure " + e.getLocalizedMessage());
                    }

                    @Override
                    public void onResponse(Call call, Response response) throws IOException {
                        LogUtils.i(TAG, "onResponse " + response.body().string());
                    }
                });
}
```

### 2.2源码分析

**第一步，创建okhttpclient对象：**

```
OkHttpClient mClient = new OkHttpClient();
```

源码：

```
 public OkHttpClient() {
    this(new Builder());
  }

  OkHttpClient(Builder builder) {
    this.dispatcher = builder.dispatcher;
    this.proxy = builder.proxy;
   ....
}
```

这里使用了builder模式，创建各种参数

**第二步，创建http请求：**

```
   Request request = new Request.Builder().url(finalUrl).build();
```

源码：

```
public final class Request {
	...
  Request(Builder builder) {
    this.url = builder.url;
    this.method = builder.method;
    this.headers = builder.headers.build();
    this.body = builder.body;
    this.tags = Util.immutableMap(builder.tags);
  }

  public static class Builder {
    @Nullable HttpUrl url;
    String method;
   
    public Builder() {
      this.method = "GET";
      this.headers = new Headers.Builder();
    }

    public Builder url(HttpUrl url) {
      if (url == null) throw new NullPointerException("url == null");
      this.url = url;
      return this;
    }
    
   }
     public Request build() {
      if (url == null) throw new IllegalStateException("url == null");
      return new Request(this);
    }
}

```

**第三步，发送http请求：**

```
 mClient.newCall(request).enqueue(new Callback() {
                    @Override
                    public void onFailure(Call call, IOException e) {
                        LogUtils.i(TAG, "onFailure " + e.getLocalizedMessage());
                    }

                    @Override
                    public void onResponse(Call call, Response response) throws IOException {
                        LogUtils.i(TAG, "onResponse " + response.body().string());
                    }
                });
```

源码：

```
/**
 * Prepares the {@code request} to be executed at some point in the future.
 */
@Override public Call newCall(Request request) {
  return RealCall.newRealCall(this, request, false /* for web socket */);
}

 private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    this.client = client;
    this.originalRequest = originalRequest;
    this.forWebSocket = forWebSocket;
    this.retryAndFollowUpInterceptor = new RetryAndFollowUpInterceptor(client, forWebSocket);
    this.timeout = new AsyncTimeout() {
      @Override protected void timedOut() {
        cancel();
      }
    };
    this.timeout.timeout(client.callTimeoutMillis(), MILLISECONDS);
  }

  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }

```

这里创建了RealCall

![image-20181213144050803](https://ws3.sinaimg.cn/large/006tNbRwgy1fy545mz3fdj30wc074wgc.jpg)

利用` client.dispatcher().enqueue(new AsyncCall(responseCallback));` 来进行网络请求，

`AsyncCall`是`RealCall`的内部类，继承了`NamedRunnable`类

![image-20181213144147184](https://ws2.sinaimg.cn/large/006tNbRwgy1fy545nwntyj313i0i0tc7.jpg)
来看一下`AsyncCall`的`execute`方法：

![image-20181213144311495](https://ws4.sinaimg.cn/large/006tNbRwgy1fy545nkvwjj311r0u0qa9.jpg)

核心逻辑在这`  Response response = getResponseWithInterceptorChain();`，返回后，回调`onResponse`或`onFailure`,并在finally时，关闭当前请求。

#### 2.2.1 Dispatcher介绍：

我们先回去 看一下`client.dispatcher()`

```
 public Dispatcher dispatcher() {
    return dispatcher;
  }
   public Builder() {
      dispatcher = new Dispatcher();
  }
```

`dispatcher`是`Okhttpclient.Builder`的成员变量

```
public final class Dispatcher {
  private int maxRequests = 64;  //最大并发数
  private int maxRequestsPerHost = 5; //每个主机最大并发数
  private @Nullable Runnable idleCallback;

  /** Executes calls. Created lazily. */ 线程池
  private @Nullable ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  /**准备执行的异步请求*/
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  /** 正在执行的异步请求，包含已经取消但是未finish的请求 */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  /** 正在执行的同步请求，包含已经取消但是未finish的请求 */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }
   public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
  。。
}
```

构建了一个核心为[0, Integer.MAX_VALUE]的线程池，它不保留任何最小线程数，随时创建更多的线程数，当线程空闲时只能活60秒，它使用了一个不存储元素的阻塞工作队列，一个叫做"OkHttp Dispatcher"的线程工厂。

也就是说，在实际运行中，当收到10个并发请求时，线程池会创建十个线程，当工作完成后，线程池会在60s后相继关闭所有线程。

```
 void enqueue(AsyncCall call) {
    synchronized (this) {
      readyAsyncCalls.add(call);
    }
    promoteAndExecute();
  }
  
```

当创建异步请求时，加入到准备队列`readyAsyncCalls`,执行`promoteAndExecute`方法：

```
   /**
   * Promotes eligible calls from {@link #readyAsyncCalls} to {@link #runningAsyncCalls} and runs
   * them on the executor service. Must not be called with synchronization because executing calls
   * can call into user code.
   *
   * @return true if the dispatcher is currently running calls.
   */
  private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();

        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        if (runningCallsForHost(asyncCall) >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        executableCalls.add(asyncCall);
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }

    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      asyncCall.executeOn(executorService());
    }

    return isRunning;
  }
```

开始遍历准备队列，如果没有超过最大并发数并且没超过单主机并发数时，加入正在进行的队列，调用线程池执行请求；超过时，继续准备。

```
  /** Returns the number of running calls that share a host with {@code call}. */
  private int runningCallsForHost(AsyncCall call) {
    int result = 0;
    for (AsyncCall c : runningAsyncCalls) {
      if (c.get().forWebSocket) continue;
      if (c.host().equals(call.host())) result++;
    }
    return result;
  }
```

通过遍历计数判断相同主机的并发数。

#### 2.2.2  client.dispatcher().finished

```
 try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        e = timeoutExit(e);
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
```

这里看到，执行结束后，必然执行`client.dispatcher().finished(this)`

```
 /** Used by {@code AsyncCall#run} to signal completion. */
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call);
  }

  /** Used by {@code Call#execute} to signal completion. */
  void finished(RealCall call) {
    finished(runningSyncCalls, call);
  }

private <T> void finished(Deque<T> calls, T call) {
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      idleCallback = this.idleCallback;
    }

    boolean isRunning = promoteAndExecute();

    if (!isRunning && idleCallback != null) {
      idleCallback.run();
    }
  }
```

这里有三个方法，最终调用的时候，又调用`promoteAndExecute`.遍历准备队列，如果没有超过最大并发数并且没超过单主机并发数时，加入正在进行的队列，调用线程池执行请求；超过时，继续准备。

#### 2.2.3  getResponseWithInterceptorChain

![image-20181203111615626](https://ws2.sinaimg.cn/large/006tNbRwgy1fxtmrq4t3sj31560eetdp.jpg)

![image-20181203201408803](https://ws2.sinaimg.cn/large/006tNbRwgy1fy1p1w1yz9j30wi02saaq.jpg)

1）在配置 OkHttpClient 时设置的 interceptors；
 2）负责失败重试以及重定向的 RetryAndFollowUpInterceptor；
 3）负责把用户构造的请求转换为发送到服务器的请求、把服务器返回的响应转换为用户友好的响应的 BridgeInterceptor；
 4）负责读取缓存直接返回、更新缓存的 CacheInterceptor；
 5）负责和服务器建立连接的 ConnectInterceptor；
 6）配置 OkHttpClient 时设置的 networkInterceptors；
 7）负责向服务器发送请求数据、从服务器读取响应数据的 CallServerInterceptor。

从上图可以看出Okhttp添加拦截器有两个位置：
1、调用OkhttpClient对象addInterceptor方法添加的拦截器集合，会添加到拦截器链的顶部位置。
2、调用OkhttpClient对象addNetworkInterceptor方法添加的拦截器集合，会将这些拦截器插入到ConnectInterceptor和CallServiceInterceptor两个拦截器的中间。

这两种自定义拦截器的区别就是：通过addInterceptor添加的拦截器可以不需要调用proceed方法。
而addNetwordInterceptor的拦截器则必须拦截器链的procceed方法，以确保CallServerInterceptor拦截器的执行。

1）应用拦截器

- 不需要担心中间过程的响应,如重定向和重试.

- 总是只调用一次,即使HTTP响应是从缓存中获取.

- 观察应用程序的初衷. 不关心OkHttp注入的头信息如: If-None-Match.

- 允许短路而不调用 Chain.proceed(),即中止调用.

- 允许重试,使 Chain.proceed()调用多次.

2）网络拦截器

- 能够操作中间过程的响应,如重定向和重试.
- 当网络短路而返回缓存响应时不被调用.
- 只观察在网络上传输的数据.
- 携带请求来访问连接.

OkHttp的这种拦截器链采用的是责任链模式，这样的好处是将请求的发送和处理分开，并且可以动态添加中间的处理方实现对请求的处理、短路等操作。

从上述源码得知，不管okhttp有多少拦截器最后都会走，如下方法：![image-20181203111634777](https://ws3.sinaimg.cn/large/006tNbRwgy1fxtmro0c4pj313u042q48.jpg)

Interceptor 代码如下：

![image-20181203111713560](https://ws4.sinaimg.cn/large/006tNbRwgy1fxtmrpkvd1j316g0i2420.jpg)

看一下`RealInterceptorChain`的具体实现：

![image-20181203111826126](https://ws4.sinaimg.cn/large/006tNbRwgy1fxtmrp3fnvj31400g2dkm.jpg)

![image-20181203111516257](https://ws2.sinaimg.cn/large/006tNbRwgy1fxtmrndtvtj315x0u0qce.jpg)

这里创建`RealInterceptorChain`的实例`next`,取得`interceptors`的第`index`个元素，调用`intercept方法`，这里把`next`,后续的`interceptor`同样会调用`RealInterceptorChain`的`proceed`方法，行成链式操作。

![è¿éåå¾çæè¿°](https://ws3.sinaimg.cn/large/006tNbRwgy1fxtmrqy9f6j314j0iztc7.jpg)

##### 2.2.3.1 RetryAndFollowUpInterceptor

![image-20181203171514962](https://ws4.sinaimg.cn/large/006tNbRwgy1fy1p1v22c5j30wi02saaq.jpg)

执行StreamAllocation创建时，可以看到根据客户端请求的地址url，还调用了createAddress方法。进入该方法可以看出，这里返回了一个创建成功的Address，实际上Address就是将客户端请求的网络地址，以及服务器的相关信息，进行了统一的包装，也就是将客户端请求的数据，转换为OkHttp框架中所定义的服务器规范，这样一来，OkHttp框架就可以根据这个规范来与服务器之间进行请求分发了。关于createAddress方法的源代码如下所示，

![image-20181203172128916](https://ws3.sinaimg.cn/large/006tNbRwgy1fy1p1vds2wj312k0cewik.jpg)

们分析StreamAllocation的构造方法，观察此构造方法可以看出，StreamAllocation创建时，又根据Address，创建了一个路由选择器RouteSelector，可以这样理解，OkHttp通过层层的封装，将请求的服务器相关信息，封装到了不同层次中，而每个层次所处理的职责不同，而Address、RouteSelector等，都是不同层次中，用于对服务器相关信息的包装处理，同时也起到了桥梁的作用。关于StreamAllocation的源代码，如下所示。 

![image-20181203172447334](https://ws1.sinaimg.cn/large/006tNbRwgy1fy1p1z5j9jj313s05y0ug.jpg)

![image-20181203172434261](https://ws3.sinaimg.cn/large/006tNbRwgy1fy1p3f2ar5j30vg062myd.jpg)

![image-20181203172544805](https://ws1.sinaimg.cn/large/006tNbRwgy1fy1p3e9qusj30vm0bytbx.jpg)

接下来继续回到RetryAndFollowUpInterceptor，这时会进入一个while循环，在这个循环中，首先会进行一个安全检查操作，检查当前请求是否被取消，如果这时请求被取消了，则会通过StreamAllocation释放连接，并抛出异常，源代码如下所示，

![image-20181203172627863](https://ws3.sinaimg.cn/large/006tNbRwgy1fy1p24te9rj30my05awf2.jpg)

如果这时没有发生上述情况，接下来会通过RealInterceptorChain的proceed方法处理请求，在请求过程中，只要发生异常，releaseConnection就会为true，一旦变为true，就会将StreamAllocation释放掉，源代码如下所示：

![image-20181203172710946](https://ws4.sinaimg.cn/large/006tNbRwgy1fy1p1zqzr5j31aw0li0yh.jpg)

如果这时没有产生异常情况，接下来则会通过响应Response来执行followUpRequest方法（这时所有的Interceptor已经执行完，得到了response），来检查是否需要进行重定向操作。这里我们暂且先不分析此方法，继续在while循环中向下分析，当不需要进行重新定向操作时，就会直接返回Response，源代码如下所示。,

![image-20181203172834045](https://ws3.sinaimg.cn/large/006tNbRwgy1fy1p45s3blj30mw08a3zf.jpg)

这些都是目前正常情况下，RetryAndFollowUpInterceptor重定向机制的处理流程，接下来我们分析当请求发生异常时，RetryAndFollowUpInterceptor是如何处理重定向操作的。 
如果这时需要进行请求重定向操作，那么此时followUp这个请求头就不会为空，先看一下followUpRequest方法：

![image-20181203193204447](https://ws3.sinaimg.cn/large/006tNbRwgy1fy1p44npddj30z10u0qcx.jpg)

根据响应码responseCode来检查当前是否需要进行请求重定向，我们知道，在Http响应码中，处于3XX的，都需要进行请求重定向处理。因此，接下来该方法会通过switch…case…来进行不同的响应码处理操作。 

接下来首先对重定向请求进行了两个安全检查，代码如下所示:

![image-20181203172946647](https://ws1.sinaimg.cn/large/006tNbRwgy1fy1p20rvrpj312k0hojwe.jpg)

![image-20181203173205968](https://ws2.sinaimg.cn/large/006tNbRwgy1fy1p43yyfwj313q08y40j.jpg)

然后根据重定向请求followUp，与当前的响应进行对比，检查是否同一个连接。通常，当发生请求重定向时，url地址将会有所不同，也就是说，请求的资源在这时已经被分配了新的url。因此，接下来的!sameConnection这个判断将会符合，该这个方法就是用来检查重定向请求，和当前的请求，是否为同一个连接。一般来说，客户端进行重定向请求时，需要与新的url建立连接，而原先的连接，则需要进行销毁,然后，根据重定向请求获取到的新的url，再次创建了一个新的StreamAllocation连接。

在最后，还会看到这样一行代码，就是将重定向得到的新请求followUp，赋值给了request，这样，在while循环中，就可以根据根据重定向操作，重新生成的这个新的Request，继续向客户端发送请求获取数据了，代码如下所示

![image-20181203193722943](https://ws3.sinaimg.cn/large/006tNbRwgy1fy1p1wqc5rj30be032aa8.jpg)



##### 2.2.3.2 BridgeInterceptor

```
/**
 * Bridges from application code to network code. First it builds a network request from a user
 * request. Then it proceeds to call the network. Finally it builds a user response from the network
 * response.
```

BridgeInterceptor从用户的请求构建网络请求，然后提交给网络，最后从网络响应中提取出用户响应。

![image-20181203200145859](https://ws4.sinaimg.cn/large/006tNbRwgy1fy1p42bs6rj30y30u0qdn.jpg)

这里创建http请求的header

![image-20181203201010941](https://ws4.sinaimg.cn/large/006tNbRwgy1fy1p432vlwj314u0igte4.jpg)

然后调用后续的interceptor进行网络请求，并处理response

##### 2.2.3.3.CacheInterceptor

正如浏览器在访问网络请求的时候提供缓存功能一样，Okhttp也提供了缓存功能

在服务器响应的时候，在”响应头信息”里面包含了资源的最后修改时间，我们可以通过Last-Modified这个header来获取该时间： 

![è¿éåå¾çæè¿°](https://ws4.sinaimg.cn/large/006tNbRwgy1fy1p243rhvj30mq0f1jtr.jpg)

既然我们能拿到资源文件的最后修改时间，那么我们就可以根据这个时间来判断服务器上面的资源是否有修改了！怎么做呢？当我们再次请求服务器的时候，在”请求头信息”会带上次请求获取的Last-Modified时间，该时间在请求头信息”用If-Modified-Since 这个header里面，见下图：

![è¿éåå¾çæè¿°](https://ws4.sinaimg.cn/large/006tNbRwgy1fy1p3fmietj30pn0ejgoe.jpg)

当服务器收到请求后检测到If-Modified-Since这个Header,则与被请求资源的最后修改时间进行比对：

```
if(服务器资源最后修改时间<=If-Modified-Since) {
  说明对应的资源没有修改过，此时服务器返回304状态码，不再需要将报文主体部分返回给客户端；
  客户端此时直接用缓存的资源即可
}else {
   说明资源又被改动过，服务器返回状态码200，此时就需要将报文的主体部分返给客户端，；
}
```

其实，通过上面的图我们发现在请求/响应的头信息里面还有两个Header:If-None-Match/Etag。当第一次请求的时候，”响应头信息”里面有一个Etag的Header可以看做是资源的标识符。再次请求的时候，”请求头信息”会包含一个If-None-Match的头信息。此时服务器取得If-None-Match后会和资源的Etag进行比如，如果相同则说明资源没改动过，那么响应304，客户端可以使用缓存；否则返回200，并且将报文的主题返回给客户端

**Last-Modified & If-Modified-Since **

**Last-Modified与Etag类似。不过Last-Modified表示响应资源在服务器最后修改时间而已。与Etag相比，不足为：**

　　- **Last-Modified标注的最后修改只能精确到秒级，如果某些文件在1秒钟以内，被修改多次的话，它将不能准确标注文件的修改时间；**

　　- **如果某些文件会被定期生成，当有时内容并没有任何变化，但Last-Modified却改变了，导致文件没法使用缓存；**

　　- **有可能存在服务器没有准确获取文件修改时间，或者与代理服务器时间不一致等情形。**

**然而，Etag是服务器自动生成或者由开发者生成的对应资源在服务器端的唯一标识符，能够更加准确的控制缓存。Etag的优先级高于Last-Modified。**

下边看一下`CacheInterceptor`的`interceptor`方法：

![image-20181209104827001](https://ws3.sinaimg.cn/large/006tNbRwgy1fy1p1xeyyxj30u40u0ak9.jpg)首先从cache中获取当前request的response

![image-20181209172305355](https://ws3.sinaimg.cn/large/006tNbRwgy1fy1p3ghpefj30i4036q3a.jpg)

这个cache是`InternalCache`的实例，在`RealCall`类中：

```
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
   ....
    interceptors.add(new CacheInterceptor(client.internalCache()));
    .....
    return chain.proceed(originalRequest);
  }
```

这个cache是从`okhttpClient`中取得的，那么肯定可以通过`OkhttpClient.Builder`进行构造：

```
File httpCacheDirectory = new File(AppContext.context.getCacheDir(), "okhttpCache");
    int cacheSize = 10 * 1024 * 1024; // 10 MiB
    Cache cache = new Cache(httpCacheDirectory, cacheSize);
    OkHttpClient client = new OkHttpClient.Builder()
            .addNetworkInterceptor(NetCacheInterceptor)
            .addInterceptor(OfflineCacheInterceptor)
            .cache(cache)
            .connectTimeout(10, TimeUnit.SECONDS)
            .readTimeout(10, TimeUnit.SECONDS)
            .build();
```

我们在回到`CacheStratey`这里来：

![image-20181209172213153](https://ws1.sinaimg.cn/large/006tNbRwgy1fy1p3g3be2j314w07gwgh.jpg)这里取得某种缓存策略，暂时不多讲。
![image-20181209172201150](https://ws4.sinaimg.cn/large/006tNbRwgy1fy1p21phq4j310i0ju78t.jpg)

如果缓存为空，并且禁止使用网络，返回空的response,响应码504；如果只是禁止使用网络，返回cache的reponse。

![image-20181209172508053](https://ws2.sinaimg.cn/large/006tNbRwgy1fy1p3doux6j30wc094gnc.jpg)

下边就继续走interceptor，进行网络请求。

![image-20181209172622660](https://ws4.sinaimg.cn/large/006tNbRwgy1fy1p1yhi5bj30wq0gqgqv.jpg)

如果返回网络请求了，并且响应码为304，直接读取缓存中的reponse。

![image-20181209172843735](https://ws1.sinaimg.cn/large/006tNbRwgy1fy1p22xmhej313i0gmq6u.jpg)

如果不是返回304，直接使用返回的数据，并且缓存response.

**这里注意的是，只要允许使用网络，okhttp都会进行网络请求，只是服务器如果没有更新（返回304）时，才从cache中读取**

##### 2.2.3.4.ConnectInterceptor

![image-20181210154408149](https://ws3.sinaimg.cn/large/006tNbRwgy1fy1p45cm83j311e0asgpl.jpg)

从代码上来看该拦截器的主要功能都交给了`StreamAllocation`处理，且这个类是从拦截器链对象（RealInterceptorChain对象）上获取的，我们知道RealInterceptorChain是在RealCall的getResponseWithInterceptorChain方法初始化的

![image-20181210154805214](https://ws1.sinaimg.cn/large/006tNbRwgy1fy545s7recj31hs0getel.jpg)

这里会发现，streamAllocation参数传的是null,我们知道这个是链式操作，可能在某个Interceptor中，传入了值，去看看RetryAndFollowUpInterceptor:

![image-20181210154939130](https://ws3.sinaimg.cn/large/006tNbRwgy1fy545rqsgqj31760e6gp0.jpg)

果然是在这里，也就是每次请求都需要创建一个`StreamAllocation`对象，该构造器里面有两个重要的参数： 
1.使用了Okhttp的连接池ConnectionPool 
2.通过url创建了一个Address对象。 

在Okhttp内部的连接池实现类为ConnectionPool，该类持有一个ArrayDeque队列作为缓存池，该队列里的元素为RealConnection(通过这个名字应该不难猜出RealConnection是来干嘛的）。

该链接池在初始化OkhttpClient对象的时候由OkhttpClient的Builder类创建，并且ConnectionPool提供了put、get、evictAll等操作。但是Okhttp并没有直接对连接池进行获取，插入等操作；而是专门提供了一个叫Internal的抽象类来操作缓冲池：比如向缓冲池里面put一个RealConnection，从缓冲池get一个RealConnection对象，该类里面有一个public且为static的Internal类型的引用：

![image-20181210163728135](https://ws4.sinaimg.cn/large/006tNbRwgy1fy545ol4y2j314a0k0gqt.jpg)

我们再回到`ConnectInterceptor`:

![image-20181217135329671](https://ws2.sinaimg.cn/large/006tNbRwgy1fy9sg2o9lxj310q0akn13.jpg)

首先`streamAllocation`调用`newStream`方法,newStream方法主要做了工作：

1）从缓冲池ConnectionPool获取一个RealConnection对象，如果缓冲池里面没有就创建一个RealConnection对象并且放入缓冲池中，具体的说是放入ConnectionPool的ArrayDeque队列中。

2）获取RealConnection对象后并调用其connect**打开Socket链接

![image-20181210164318511](https://ws3.sinaimg.cn/large/006tNbRwgy1fy545m86qzj31560he43w.jpg)

1、调用findHealthyConnection获取一个RealConnection对象。 
2、通过获取到的RealConnection来生成一个HttpCodec对象并返回。

先看下`findHealthyConnection `：

![image-20181210164415003](https://ws4.sinaimg.cn/large/006tNbRwgy1fy545mkfe5j31760ns447.jpg)

1、开启一个while循环，调用findConnection继续获取RealConnection对象candidate 。 
2、如果candidate 的successCount 为0，直接返回之，while循环结束 
3、如果candidate是一个不“健康”的对象，则对此对象进行调用noNewStreams进行销毁处理，继续循环调用findConnection获取RealConnection对象。

（注：不健康的RealConnection条件为如下几种情况： 
RealConnection对象 socket没有关闭 
socket的输入流没有关闭 
socket的输出流没有关闭 
http2时连接没有关闭 ）

继续看`findConnection`:

![image-20181210164624353](https://ws1.sinaimg.cn/large/006tNbRwgy1fy545qvt99j30zb0u0wnr.jpg)

![image-20181210164655562](https://ws1.sinaimg.cn/large/006tNbRwgy1fy545qepeaj30x60u0dnr.jpg)

![image-20181210164717374](https://ws1.sinaimg.cn/large/006tNbRwgy1fy545pleilj310s0u0wm2.jpg)

1、StreamAllocation的connection能复用就复用之 
2、如果connection不能复用，则从连接池中获取RealConnection，获取成功则返回，从连接池中获取RealConnection的方法调用了两次 
,第一次没有传Route,第二次传了
3，如果连接池里没有则neｗ一个RealConnection对象，并放入连接池中 
4.最终调用RealConnection的connect方法打开一个socket链接

##### 2.2.3.5.CallServerInterceptor

CallServerInterceptor的主要功能就是—**向服务器发送请求，并最终返回Response对象供客户端使用。**

![image-20181217134824910](https://ws4.sinaimg.cn/large/006tNbRwgy1fy9sfuz3s7j313s0sc12z.jpg)

![image-20181217134934619](https://ws4.sinaimg.cn/large/006tNbRwgy1fy9sfsgj0pj312i0u0ahl.jpg)

该方法首先是获取了httpCodec对象，该对象的主要功能就是**对不同http协议（http1.1和http/2）的请求和响应做处理**，该对象的初始化是在ConnectIntercepor的intercept里面：

```
HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
```

![image-20181217135819710](https://ws4.sinaimg.cn/large/006tNbRwgy1fy9sfxwefcj314m0g0dkp.jpg)

`RealConnection`调用了`newCodec`方法：

![image-20181217140055497](https://ws3.sinaimg.cn/large/006tNbRwgy1fy9sfvm1jpj30ze09w0vz.jpg)

我们知道Http发送网络请求前两个步骤是： 
1、建立TCP链接 
2、客户端向web服务器发送请求命令：形如GET /login/login.jsp?username=android&password=123 HTTP/1.1的信息

在Okhttp中ConnectInterceptor负责第一个步骤，那么第二个步骤是如何实现的呢？答案就是httpCodec对象的writeRequestHeaders方法：

![image-20181217142318823](https://ws3.sinaimg.cn/large/006tNbRwgy1fy9sfxhn0zj313c0d0wif.jpg)

![image-20181217142347070](https://ws4.sinaimg.cn/large/006tNbRwgy1fy9sg0r8mbj310s0c6dj2.jpg)

这里使用okio的sink写入数据。

我们知道HTTP支持post,delete,get,put等方法，而post，put等方法是需要请求体的（在Okhttp中用RequestBody来表示）。所以接着writeRequestHeaders之后Okhttp对请求体也做了响应的处理：

![image-20181217145205649](https://ws4.sinaimg.cn/large/006tNbRwgy1fy9sfzb0ufj313608qwhs.jpg)

1.`HttpMethod`调用`permitRequestBody`判断是否需要请求body:

![image-20181217145300763](https://ws2.sinaimg.cn/large/006tNbRwgy1fy9sft9ploj31340e0jvs.jpg)

2.下一步对`Except`头部进行了判断：

![image-20181217145430828](https://ws1.sinaimg.cn/large/006tNbRwgy1fy9sftlsg9j312i0u0ahl.jpg)

![image-20181217145549015](https://ws4.sinaimg.cn/large/006tNbRwgy1fy9sfzubzlj31ae0n0q8x.jpg)

如果返回100，说明接受body,返回null;

![image-20181217153335809](https://ws3.sinaimg.cn/large/006tNbRwgy1fy9sfu1u3jj31q40ii0zr.jpg)

接下来就写入body数据。，客户端向服务端发送请求的部分已经结束，下面就剩下读取服务器响应然后构建Response对象了：

![image-20181217153742000](https://ws1.sinaimg.cn/large/006tNbRwgy1fy9sg1eggoj314j0u0ah6.jpg)

1.调用httpcodec的readResponseHeaders:

![image-20181217145549015](https://ws4.sinaimg.cn/large/006tNbRwgy1fy9sfzubzlj31ae0n0q8x.jpg)

之前处理`Expect`的时候已经看过，这里构造ResponseBuilder对象

2.构造reponse:

![image-20181217154058765](https://ws3.sinaimg.cn/large/006tNbRwgy1fy9sfyf9fnj31580fstcg.jpg)

这里，body是用`httpCodec.openResponseBody`返回的：

![image-20181217154155926](https://ws3.sinaimg.cn/large/006tNbRwgy1fy9sfw3nphj313g04udha.jpg)

![image-20181217154546903](https://ws3.sinaimg.cn/large/006tNbRwgy1fy9sfyyncsj311w0fm42g.jpg)

最终的返回数据为RealReponseBody对象，它提供了string方法：

![image-20181217154433701](https://ws2.sinaimg.cn/large/006tNbRwgy1fy9sg09jdlj318i0gmtdd.jpg)

string(）方法也很简单，就是通过一些处理然后让调用source.readString来读取服务器的数据。需要注意的是该方法最后调用closeQuietly来关闭了当前请求的InputStream输入流，所以string()方法只能调用一次,再次调用的话会报错，毕竟输入流已经关闭了

到此为止Okhttp从发起请求到响应请求生成Response对象的流程已经分析完毕