---
layout: post
title: "anatomy-of-android-debug-hwui-profile"
date: 2016-10-18 20:21:32 +0800
comments: true
categories: 
keywords: 
description: 
#published: false
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

### dump

### 统计