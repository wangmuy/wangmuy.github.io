---
layout: post
title: "anatomy-of-systemui-recents"
date: 2017-01-04 14:38:43 +0800
comments: true
categories: 
keywords: 
description: 
#published: false
---

版本: android-6.0.0_r1

# 初始化
RecentsTaskLoader.initialize() {
  初始化 mApplicationIconCache/mThumbnailCache/mActivityLabelCache/mContentDescriptionCache
  mLoadQueue = new TaskResourceLoadQueue()
  new TaskResourceLoader(mLoadQueue) {用HandlerThread起LoaderThread, post自身(Runnable)} }
<- Recents.start() systemui启动时各模块初始化调用

# TaskResourceLoader.mLoadThread 加载线程
逐个加载推到 mLoadQueue的 Task的相关内容 icon/thumbnail/label
加载完调用 (Task t).notifyTaskDataLoaded(thumbnail, icon) 通知更新

# 加载Task

## 加载所有task

每次加载所有task都使用一个 plan

```
1. RecentTaskLoader.loadTasks(ctx, plan, opts)
opts.numVisibleTasks 控制plan加载时是否加载icon, opts.numVisibleTaskThumbnails 控制plan加载时是否加载thumbnail
1.1. <- Recents.start() 最先调用
1.2. <- RecentsActivity.updateRecentsTasks() 启动最近应用时调用, 消费Recents.sInstanceLoadPlan
  <- RecentsActivity.onStart()
1.3. <- Recents.preloadIcon()
  1.3.1. <- Recents.getThumbnailTransitionActivityOptions() 没有thumbnailTransitionBitmapCache或不匹配时
    <- Recents.startRecentsActivity() if(useThumbnailTransition) 获取用来切换acitivity用
  1.3.2. <- Recents.preCacheThumbnailTransitionBitmapAsync()
    <- Recents.preloadRecentsInternal() { 预加载 sInstanceLoadPlan(将来被RecentsActivity消费); if(topTask有变化) preCacheThumbnailTransitionBitmapAsync() }
    <- Recents.preloadRecents() if user是owner
    <- BaseStatusBar.preloadRecents()
    1.3.2.1 <- BaseStatusBar.mRecentsPreloadOnTouchListener() navBar的最近应用按钮响应
    1.3.2.2 <- (Handler H) BaseStatusBar.preloadRecentApps() 用于响应IstatusBar binder调用
1.4. <- TaskStackListenerImpl.run()
```

## 加载单个task

```
2. RecentTaskLoader.loadTaskData(Task t) {
    requiresLoad = (applicationIcon==null 或 thumbnail==null)
    if(requiresLoad) mLoadQueue.addTask(t)
    t.notifyTaskDataLoaded() 先通知更新, 可能为默认图, 后面LoaderThread加载完还有通知
}
<- TaskStackView.prepareViewToLeavePool(taskView, task, isNewView)
<- ViewPool.pickUpViewFromPool()
2.1. <- TaskStackView.synchronizeStackViewsWithModel()
  2.1.1 TaskStackView.computeScroll()
  2.1.2 TaskStackView.onMeasure() { if(mAwaitingFirstLayout) synchronizeStackViewsWithModel() }
  2处都是view的函数重载
2.2. <- TaskStackViewFilterAlgorithm.getEnterTransformsForFilterAnimation()
```

## taskStack 变化监听

```
注册
Recents.start() {
  mSystemServiceProxy.registerTaskStackListener(new TaskStackListenerImpl())
}
-> SystemServiceProxy.registerTaskStackListener(l) { mIam.registerTaskStackListener(l) }

变化
TaskStackListenerImpl.onTaskStackChanged() { mHandler.post(this) }
--SystemUIService主线程handler--> TaskStackListenerImpl.run()
```


## 保存/加载 taskRecord

```
TaskRecord.saveToXml()

TaskRecord.restoreFromXml() { ActivityRecord.restoreFromXml() }
```

## 启动

AMS.startActivityFromRecents()
-> AMS.startActivityFromRecentsInner()

## 截屏大小

frameworks/base/core/res/res/values-sw720dp/dimens.xml 添加
```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <dimen name="thumbnail_width">1280dp</dimen>
    <dimen name="thumbnail_height">720dp</dimen>
</resources>
```

## 长按启动

frameworks/base/core/res/res/values-sw720dp/config.xml 添加

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <integer name="config_longPressOnHomeBehavior">1</integer>
</resources>
```