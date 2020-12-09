---
title:      "React Native是如何工作的" 
date:       2020-03-07 22:00:00
tags:
    - React Native
categories:
    - React Native
---

## React Native是如何工作的？

### React Native App中的线程

在React Native App中有4个线程：

    1）UI线程：即主线程。它用于Android和iOS原生UI的渲染。比如在安卓上，关于UI的测量/布局/绘制就是在UI线程上。

    2）JS线程：JS线程或JavaScript线程是逻辑将会在其中运行。比如应用程序的JavaScript代码执行、API调用、触摸事件处理都是在这个线程中运行的。当JS线程中的每个事件循环结束时，对于UI的更新将被分批发送到native端进行更新。

    为了拥有更好的性能，JS线程应该能够在下一帧渲染前将更新内容传递给UI线程。比如iOS每秒钟渲染60帧，每一帧间隔为16.67ms。如果在JS线程中的事件循环内做了比较复杂的操作花费大于16.67ms，UI将会卡顿。

    有一个例外是，如果UI本来就在native端发生变化，则不会由于JS端阻塞而卡顿。

    3）Native Modules线程：有时候App需要访问native端的API，则会在Native Module线程中调用。

    4）渲染线程：仅在Android L (5.0)时，渲染线程用于生成OpenGL命令去绘制UI。

### 在程序过程中React Native是如何工作的

    1）在程序启动时，在主线程中开始解析并加载`JS Bundle`

    2) 当JavaScript代码加载成功之后，主线程把JavaScript代码送到另一个JS线程，这样当JS代码在做一些繁重的计算任务导致阻塞一段时间也不会影响UI线程。

    3）当React开始渲染，Reconciler开始计算差异，同时会生成一个新的虚拟DOM(布局)，它把这些DOM上的变化传递给另一个Shadow Thread（为什么叫做“Shadow”，因为他会生成“Shadow Node”）。

    4）Shadow Thread计算布局然后把布局参数或布局对象传递给主线程。

    5）由于只有主线程才能在屏幕上进行渲染，所以Shadow Thread把生成好的布局传递给主线程，然后渲染UI。

### React Native的组成

通常我们把React Native分成3部分：

    1）React Native - 原生端即Native

    2）React Native - JS端

    3）React Native - 桥即bridge

翻译自[How React Native works?](https://www.geeksforgeeks.org/react-native-works/)
