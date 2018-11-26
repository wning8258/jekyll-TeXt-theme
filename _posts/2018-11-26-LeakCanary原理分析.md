---
layout: post
title: LeakCanary原理分析
key: 20181126 LeakCanary原理分析
tags: LeakCanary WeakReference
---



LeakCanary是Square为Android应用提供的一个监测内存泄露的工具，源码地址：<https://github.com/square/leakcanary>。

<!--more-->

# 1.使用

在gradle文件中引入依赖：

```
dependencies {
    debugCompile 'com.squareup.leakcanary:leakcanary-android:1.5.4'
    releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.5.4'
}
```

在项目的Application中添加检测：

```
public class MainApplication extends Application {

  @Override public void onCreate() {
    super.onCreate();
    if (LeakCanary.isInAnalyzerProcess(this)) {
      // This process is dedicated to LeakCanary for heap analysis.
      // You should not init your app in this process.
      return;
    }
    LeakCanary.install(this);
  }
}
```

如果检测到有内存泄漏，手机桌面会多出一个图标，点进去查看可以看见泄漏信息:

![img](https://ws4.sinaimg.cn/large/006tNbRwgy1fxlhwdz4obj30jg0a4gnk.jpg)

# 2.原理

我们知道LeakCanary会自动监测Activity的内存泄漏，不需要加任何代码。到底是如何自动监测的呢？

## 2.1 如何监测Activity

直接看 `LeakCanary.install(this);`

```
/**
 * Creates a {@link RefWatcher} that works out of the box, and starts watching activity
 * references (on ICS+).
 */
public static RefWatcher install(Application application) {
  return install(application, DisplayLeakService.class,
      AndroidExcludedRefs.createAppDefaults().build());
}


  /**
   * Creates a {@link RefWatcher} that reports results to the provided service, and starts watching
   * activity references (on ICS+).
   */
  public static RefWatcher install(Application application,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass,
      ExcludedRefs excludedRefs) {
    if (isInAnalyzerProcess(application)) {
      return RefWatcher.DISABLED;
    }
    enableDisplayLeakActivity(application);
    HeapDump.Listener heapDumpListener =
        new ServiceHeapDumpListener(application, listenerServiceClass);
    RefWatcher refWatcher = androidWatcher(application, heapDumpListener, excludedRefs);
    ActivityRefWatcher.installOnIcsPlus(application, refWatcher);
    return refWatcher;
  }


  /**
   * Creates a {@link RefWatcher} with a default configuration suitable for Android.
   */
  public static RefWatcher androidWatcher(Context context, HeapDump.Listener heapDumpListener,
      ExcludedRefs excludedRefs) {
    LeakDirectoryProvider leakDirectoryProvider = new DefaultLeakDirectoryProvider(context);
    DebuggerControl debuggerControl = new AndroidDebuggerControl();
    AndroidHeapDumper heapDumper = new AndroidHeapDumper(context, leakDirectoryProvider);
    heapDumper.cleanup();
    Resources resources = context.getResources();
    int watchDelayMillis = resources.getInteger(R.integer.leak_canary_watch_delay_millis);
    AndroidWatchExecutor executor = new AndroidWatchExecutor(watchDelayMillis);
    return new RefWatcher(executor, debuggerControl, GcTrigger.DEFAULT, heapDumper,
        heapDumpListener, excludedRefs);
  }


```

**第一步：**

![image-20181126154906682](https://ws2.sinaimg.cn/large/006tNbRwgy1fxliin8qflj30rs03cgmb.jpg)

![image-20181126154934823](https://ws1.sinaimg.cn/large/006tNbRwgy1fxlij45znwj313u0e4427.jpg)

这个方法的作用是设置四大组件开启或禁用，从上图传的参数看，是开启了查看内存泄漏的页面`DisplayLeakActivity`
**第二步：**

![image-20181126155259985](https://ws2.sinaimg.cn/large/006tNbRwgy1fxlimpqwlbj30sk01yjrr.jpg)

![image-20181126155452417](https://ws4.sinaimg.cn/large/006tNbRwgy1fxlioo4jfaj31280amjv8.jpg)

初始化一个ServiceHeapDumpListener，这是一个开启分析的接口实现类，类中定义了analyze方法，用于开启一个DisplayLeakService服务，从名字就可以看出，这是一个显示内存泄漏的辅助服务

**第三步：**

![image-20181126155703187](https://ws2.sinaimg.cn/large/006tNbRwgy1fxliqwidddj30y203w755.jpg)

![image-20181126155735583](https://ws2.sinaimg.cn/large/006tNbRwgy1fxlirgk702j310m08iaby.jpg)

创建`RefWatcher`类实例用于检测内存泄漏

那`RefWatcher`是什么时候检测的呢？

![image-20181126155957095](https://ws3.sinaimg.cn/large/006tNbRwgy1fxlj1g8r7fj30tw05gdgx.jpg)

![image-20181126160019287](https://ws1.sinaimg.cn/large/006tNbRwgy1fxlj1h6r5pj313w0l0q7s.jpg)

![image-20181126160040017](https://ws3.sinaimg.cn/large/006tNbRwgy1fxlj1hneicj30ww02gq3g.jpg)

调用了Application的`registerActivityLifecycleCallBacks`方法，监测Activity的生命周期方法，并在Activity销毁时，调用watch进行监测。

## 2.2 RefWatcher做了什么

 RefWatcher是一个引用检测类，它会监听可能会出现泄漏（不可达）的对象引用，如果发现该引用可能是泄漏，那么会将它的信息收集起来（HeapDumper）.

![image-20181126160703878](https://ws2.sinaimg.cn/large/006tNbRwgy1fxlj1i9ssmj314s0pgagm.jpg)

`watch`方法会为每一个`watchReference`创建一个单独的uuid key，这里的reference默认就是Activity；

创建`keyedWeakReference`实例；

在线程池中，执行`ensureGone`方法，这里是真正的监测过程。

### 2.2.1 ReferenceQueue

这里需要先讲一下监测内存泄漏的原理：

![image-20181126161314073](https://ws3.sinaimg.cn/large/006tNbRwgy1fxlk1dayllj30ty01omxq.jpg)

![image-20181126161611532](https://ws2.sinaimg.cn/large/006tNbRwgy1fxlk1i59fpj316209qn00.jpg)

![image-20181126161625208](https://ws4.sinaimg.cn/large/006tNbRwgy1fxlk1j371cj30wg0j8tcd.jpg)

`keyWeakReference`继承了`WeakReference`类，很多人都用过这个类，大多数应该都用的默认的构造方法，这里突然发现它还有另一个构造方法，传入了`ReferenceQueue`，这是干什么的呢？

这里的`queue`在`RefWatcher`实例化的时候，创建了

![image-20181126161545665](https://ws1.sinaimg.cn/large/006tNbRwgy1fxlk1hpjtuj312609gjvz.jpg)

在java的引用体系中，存在着强引用，软引用，弱引用，虚引用，这4种引用类型。对于`软引用`和`弱引用`，我们希望当一个对象被gc掉的时候通知用户线程，进行额外的处理时，就需要使用引用队列了。ReferenceQueue即这样的一个对象，当一个obj被gc掉之后，obj关联的WeakReference对象会被添加到ReferenceQueue。我们可以从queue中获取到相应的对象信息，同时进行额外的处理。比如反向操作，数据清理等。

当 GC 过后如果WeakReference对象如果一直不被加入 ReferenceQueue，它就可能存在内存泄漏。

### 2.2.2 ensureGone

![image-20181126161018141](https://ws4.sinaimg.cn/large/006tNbRwgy1fxlk1ezn4fj313m0m6gs0.jpg)

**第一步 `removeWeaklyReachableReferences`：**

![image-20181126162733730](https://ws2.sinaimg.cn/large/006tNbRwgy1fxlk1h4kgnj311o07swgh.jpg)

我们知道，当activity被回收掉之后，其引用被会加入到`ReferencePool`中，这里进行移除，如果ref不为空，证明该对象已经被回收了，没有泄露（**如果有内存泄漏的话，是无法被回收掉的，即而没有加入到ReferencePool中，这里的ref就会是空**）

![image-20181126163419340](https://ws1.sinaimg.cn/large/006tNbRwgy1fxlk1kayrej30re02uaah.jpg)

![image-20181126163430263](https://ws3.sinaimg.cn/large/006tNbRwgy1fxlk1jzfluj30me03c3z1.jpg)
集合中如果不包含这个key,证明已经被回收了。

**第二步：**

![image-20181126163633175](https://ws2.sinaimg.cn/large/006tNbRwgy1fxlk1gbxr6j312o0e4gpn.jpg)

手动调用gc，再次执行上边的方法，如果`retainedKeys`中还是包含这个key，即从来没有被回收掉，证明发生了内存泄漏，dump出hprof文件，调用`headpDumpListener.analyze`进行分析：

**第三步`headpDumpListener.analyze`：**

这里的`headpDumpListener`在`LeakCanary.install`的时候创建了：

![image-20181126163912687](https://ws4.sinaimg.cn/large/006tNbRwgy1fxlk1kwm7xj310k0bun10.jpg)

![image-20181126163934443](https://ws4.sinaimg.cn/large/006tNbRwgy1fxlk1fv0lnj31160fsgqz.jpg)

这里调用了`analyze`方法：

![image-20181126164003569](https://ws4.sinaimg.cn/large/006tNbRwgy1fxlk1ji728j30vo06yjtv.jpg)

启动了`HeapAnalyzerService`,并执行`onHandlerIntent`方法：

![image-20181126164138068](https://ws4.sinaimg.cn/large/006tNbRwgy1fxlk1dvg5sj31660bcq70.jpg)

这里调用`HeapAnalyzer`进行分析，并把结果发送给`listenerClassName`,即`DisplayLeakService`，这里的`listenerClassName`也是在`LeakCanary.install`的时候传入：

![image-20181126164624831](https://ws3.sinaimg.cn/large/006tNbRwgy1fxlpw6wpw2j30oi040dgw.jpg)

### 2.2.3 HeapAnalyzer

#### 2.2.3.1 checkForLeak

![image-20181126192727828](https://ws3.sinaimg.cn/large/006tNbRwgy1fxlpw6iwtxj315k0n67af.jpg)

**第一步：**

![image-20181126193004344](https://ws3.sinaimg.cn/large/006tNbRwgy1fxlpw8dxtsj30se054ab6.jpg)

这里使用了开源项目[haha](https://github.com/square/haha),解析hprof文件为`Snapshot`实例，解析得到的Snapshot对象直观上和我们使用MAT进行内存分析时候罗列出内存中各个对象的结构很相似，它通过对象之间的引用链关系构成了一棵树，我们可以在这个树种查询到各个对象的信息，包括它的Class对象信息、内存地址、持有的引用及被持有的引用关系等。到了这一阶段，HAHA的任务就算完成，之后LeakCanary就需要在Snapshot中找到一条有效的到被泄漏对象之间的引用路径。

**第二步:(`findLeakingReference`)**

![image-20181126193622542](https://ws4.sinaimg.cn/large/006tNbRwgy1fxlpw8twn7j30ze0ck0wx.jpg)

为了能够准确找到被泄漏对象，LeakCanary通过被泄漏对象的弱引用来在Snapshot中定位它。因为，如果一个对象被泄漏，一定也可以在内存中找到这个对象的弱引用，再通过弱引用对象的referent就可以直接定位被泄漏对象。下一步的工作就是找到一条有效的到被泄漏对象的最短的引用，这通过findLeakTrace来实现.

**第三步：（findLeakTrace）**

![image-20181126194031631](https://ws1.sinaimg.cn/large/006tNbRwgy1fxlpwcyqd1j30ue02674n.jpg)

![image-20181126194052942](https://ws1.sinaimg.cn/large/006tNbRwgy1fxlpw648j3j317c0ne7an.jpg)

 这里搜索可能内存泄漏实例到GcRoots的最短路径：

```
 ShortestPathFinder pathFinder = new ShortestPathFinder(excludedRefs);
 ShortestPathFinder.Result result = pathFinder.findPath(snapshot, leakingRef);
```

第一行创建实例，并把一些白名单的可忽视泄漏传入，`pathFinder.findPath`开始寻找最短路径：

这里通过递归寻找从GcRoots到LeakRef的最短路径，

![img](https://ws4.sinaimg.cn/large/006tNbRwgy1fxlpw9djygj30ps0n8q9s.jpg)

找到最短路径后：

![image-20181126201741462](https://ws3.sinaimg.cn/large/006tNbRwgy1fxlqb24uu9j30z802kgmd.jpg)

![image-20181126201801935](https://ws3.sinaimg.cn/large/006tNbRwgy1fxlqb2je4lj318e04uq4v.jpg)

创建`AnalysisResult`实例。

#### 2.2.3.2 sendResultToListener

![image-20181126192108011](https://ws2.sinaimg.cn/large/006tNbRwgy1fxlpwa4mguj314y01gwf8.jpg)

这里的`listenerClassName`可以一路往上追踪

![image-20181126192141820](https://ws1.sinaimg.cn/large/006tNbRwgy1fxlpwc1xb6j30q6042myb.jpg)

即`DisplayLeakService`
![image-20181126192317679](https://ws3.sinaimg.cn/large/006tNbRwgy1fxlpwckmuuj314c0bydjb.jpg)

![image-20181126192356620](https://ws1.sinaimg.cn/large/006tNbRwgy1fxlpwawcc6j310009gtb3.jpg)
这里启动了`DisplayLeakService`,并调用`onHandleIntent`方法,

![image-20181126192246120](https://ws1.sinaimg.cn/large/006tNbRwgy1fxlpwbm3gbj30t602274o.jpg)

`onHeapAnalyzed`为抽象方法，`DisplayLeakService`实现了该方法：

![image-20181126192615878](https://ws4.sinaimg.cn/large/006tNbRwgy1fxlpw7mv3oj318q0u0gx4.jpg)

这里就是把泄漏内容以notification的内容展示出来了。

