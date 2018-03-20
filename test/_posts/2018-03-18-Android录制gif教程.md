---
layout: post
title: Android录制gif教程
key: 20180318
tags: Android Gif
---

写博客的时候，有时会想录制一个gif演示效果，尝试了用vysor投影到电脑上或者用模拟器，在用licecap录制gif，发现录出来的效果不忍直视，画质感人。后来发现，Android Studio自带了视频录制功能，在使用ffmpeg把视频转成gif，效果不错。

<!--more-->

### 1.使用Android Studio录制视频

打开logcat,如图

![屏幕快照 2018-03-20 14.12.16](http://oon96myva.bkt.clouddn.com/md/3da8b.jpg)

点击左下角的按钮，就会弹出录制页面，默认会使用手机的分辨率进行录制，最长时长3分钟，什么也不填的话，会采用默认配置。点击Start Recording开始录制：

![](http://oon96myva.bkt.clouddn.com/md/h4h18.png)

点击Stop REcording即可结束录制，弹出保存对框框：

![](http://oon96myva.bkt.clouddn.com/md/7wyws.png)

录制视频就完成了

### 2.ffmpeg转gif

1. 首先安装ffmpeg

```
brew install ffmpeg
```

2. 压缩视频 （可选）

我使用的手机录制出的视频分辨率为1080*1920，展示效果的话，有时不需要那么高清，这里压缩一下分辨率：

```
ffmpeg -i test.mp4 -s 720*1280 test_out.mp4
```

3. 视频转gif

```
ffmpeg  -ss 3 -t 5 -r 15 -i test_out.mp4 test_out.gif
```

-ss 3 :从第三秒开始

-t 5:截取3秒后的5秒时长

-r 15 设定帧数

这样 gif就完成了。

看一下效果：

![](http://oon96myva.bkt.clouddn.com/md/owatc.gif)
