---
layout: post
title: "Android debug-hwui-profile"
date: 2016-10-18 20:21:32 +0800
categories: [android]
description: 
---

# 本文目标

## 探索 gpu呈现模式 测量的是什么

# 基本用法

## 原生Settings

```
Settings -> 开发者选项 -> gpu呈现模式
```

## 命令行

```bash
setprop debug.hwui.profile visual_bars # false关闭
```

# 图形解释

```
蓝色: create/update DisplayList
紫色: 同步到render thread
红色: 将DisplayList转为gl命令调用
黄色: cpu等待gpu完成
```

# profile时机

## 4.4.4_r1

### 涉及文件

```
HardwareRenderer.java
```

### 绘制

```
HardwareRenderer.drawProfileData() {
  mProfileData 数据
  for(i=0; i < getFrameCount()*elementCount; i+=elementCount)
    mProfileShapes[] <- bar条rect
    drawGraph() 画bar条
    drawCurrentFrame() if(当前帧) 加深当前帧的bar条
    drawThreadshold() 16ms水平线
```

### dump

```
dumpGfxInfo()
  mProfileData[i]   == Draw
  mProfileData[i+1] == Process
  mProfileData[i+2] == Execute
```

### 统计

#### Draw

```
buildDisplayList()
  startBuildDisplayListProfiling()
  Trace.traceBegin(TRACE_TAG_VIEW, "getDisplayList")
  view.getDisplayList()
  Trace.traceEnd(TRACE_TAG_VIEW)
  endBuildDisplayListProfiling()
```

#### Process

```
drawDisplayList()
  Trace.traceBegin(Trace.TRACE_TAG_VIEW, "drawDisplayList")
  canvas.drawDisplayList()
  Trace.traceEnd(Trace.TRACE_TAG_VIEW)
<- draw()
```

#### Execute

```
swapBuffers()
  sEgl.eglSwapBuffers(sEglDisplay, mEglSurface)
<- draw()
```

## 6.0.0_r1

### 涉及文件

```
frameworks/base/libs/hwui/renderthread/CanvasContext.cpp
frameworks/base/libs/hwui/renderthread/FrameInfoVisualizer.cpp
```

### 绘制

```
FrameInfoVisualizer::draw()
  mFrameSource[] 数据
  initializeRects() 根据数据设置所有 bar rect(left/right, top=bottom=baseline)
  drawGraph() { foreach bar: nextBarSegment() 计算bar高度; canvas->drawRects() }
  drawThreshold() 16ms水平线
```

### dump

```
FrameInfoVisulizer::dumpData()
  [IntentdedVsync, SyncStart]           == Draw
  [SyncStart, IssueDrawCommandsStart]   == Prepare
  [IssueDrawCommandsStart, SwapBuffers] == Process
  [SwapBuffers, FrameCompleted]         == Execute
```

### 流程

```
ui线程 ThreadedRenderer.java draw() -> nSyncAndDrawFrame()
--jni--> nSyncAndDrawFrame()
-> RenderProxy::syncAndDrawFrame()
-> DrawFrameTask::drawFrame() { postAndWait() 等待render线程完成当前帧 }

DrawFrameTask 继承自 RenderTask

render线程
初始化在 RenderProxy::ctor() 中 RenderThread::getInstance(), 是个Looper Thread
调用 threadLoop()
-> DrawFrameTask::run() {
  if(canUnblockUiThread) unblockUiThread() 发信号给ui线程
  if(canDrawFrame) (CanvasContext context)->draw()
  if(!canUnblockUiThread) unblockUiThread()
}
```

### 统计

#### Draw

##### IntentdedVsync

```
FrameInfo.h
UiFrameInfoBuilder::setVsync(vsyncTime, intendedVsync)
1. <- CanvasContext::doFrame() {
  setVsync() 标识vsync时间
  prepareTree()
  CanvasContext::draw() }
<- RenderThread::dispatchFrameCallbacks()
<--queue-- RenderThread::drainDisplayEventQueue()
<- RenderThread::threadLoop()

2. <- frameworks/base/core/jni android_view_Surface.cpp ContextFactory::draw()
```

##### SyncStart

```
CanvasContext::prepareTree()
  mCurrentFrameInfo->markSyncStart()
<- doFrame()
```

##### IssueDrawCommandsStart

```
CanvasContext::draw()
  mCurrentFrameInfo->markIssueDrawCommandsStart()
```

##### SwapBuffers

```
1. CanvasContext::draw()

2. FrameInfoVisualizer::draw() 临时mark当前帧
```

##### FrameCompleted

```
1. CanvasContext::draw()

2. FrameInfoVisualizer::draw() 临时mark当前帧
```