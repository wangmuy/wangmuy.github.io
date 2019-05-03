---
layout: post
title: "SystemUI 最近应用 源码分析"
date: 2017-01-04 14:38:43 +0800
categories: [android]
description: 
---

版本: android-6.0.0_r1
========================================

# 初始化

```
RecentsTaskLoader.initialize() {
  初始化 mApplicationIconCache/mThumbnailCache/mActivityLabelCache/mContentDescriptionCache
  mLoadQueue = new TaskResourceLoadQueue()
  new TaskResourceLoader(mLoadQueue) {用HandlerThread起LoaderThread, post自身(Runnable)} }
<- Recents.start() systemui启动时各模块初始化调用
```

# TaskResourceLoader.mLoadThread 加载线程
* 逐个加载推到 mLoadQueue的 Task的相关内容 icon/thumbnail/label
* 加载完调用 (Task t).notifyTaskDataLoaded(thumbnail, icon) 通知更新

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

```
AMS.startActivityFromRecents()
-> AMS.startActivityFromRecentsInner()
```

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

# activity 按 back 键所走的流程

```
按键down: Activity.onKeyDown(keyCode, event) {event.startTracking();}
按键up: Activity.onKeyUp(keyCode, event) {event.isTracking(): onBackPressed()}

Activity.onBackPressed() -> Activity.finishAfterTransition() -> Activity.finish()
-> Activity.finish(finishTask==false) 最后到 mParent==null: ActivityManagerNative.getDefault().finishActivity()
--binder--> AMS.finishActivity(token, resultCode, resultData, finishTask==false) {
  若同task仍有activity: mController.activityResuming(next.packageName)
  finishTask==false: (TaskRecord tr).stack.requestFinishActivityLocked(token, resultCode, resultData, "app-request", true) }
-> ActivityStack.finishActivityLocked((当前的 ActivityRecord r), resultCode, resultData, reason, oomAdj) {
     adjustFocusedActivityLocked(r, "finishActivity")
     finishActivityResultLocked(r, resultCode, resultData)
}
  -> ActivityStack.adjustFocusedActivityLocked(ActivityRecord r, String reason) {
    myReason = reason + " adjustFocus"
    // 当前stack是frontStack(top或可见), AMS当前焦点activity是r
    mStackSupervisor.isFrontStack(this) && (AMS mService).mFocusedActivity == r):
      ActivityRecord next = topRunningActivityLocked(null) 该stack当前的top task的top running activity, 已排除 finishing
      r.frontOfTask && task == topTask() && task.isOverHomeStack():
        if 本stack非全屏: adjustFocusedToNextVisibleStackLocked(null, myReason)
        否则 mStackSupervisor.moveHomeStackTaskToTop(task.getTaskToReturnTo(), myReason)
      重新设置到AMS
      ActivityRecord top = mStackSupervisor.topRunningActivityLocked()
      (AMS mService).setFocusedActivityLocked(top, myReason)
  }

TaskRecord.mTaskToReturnTo 表示task销毁时返回的目标, 有
* APPLICATION_ACTIVITY_TYPE=0 返回到 本task 下方的task
* HOME_ACTIVITY_TYPE=1 返回到HOME
* RECENTS_ACTIVITY_TYPE=2 返回到 recents

初始化:
field初始化为 APPLICATION_ACTIVITY_TYPE, 默认ctor初始化为 HOME_ACTIVITY_TYPE, voiceInteractor ctor保留field初始化值

TaskRecord.setTaskToReturnTo(int)
1. <- AMS.moveTaskToFrontLocked(taskId, flags, options) {
  task = mStackSupervisor.anyTaskForId(taskId); if 当前的 topRunningActivity == 最近应用: task.setTaskToReturnTo(RECENTS_ACTIVITY_TYPE) } 总结: moveTaskToFront() 时 设置返回到 最近应用
2. <- ActivityStackSupervisor.resumeHomeStackTask(homeStackTaskType, prev, reason) {
  prev != null: prev.task.setTaskToReturnTo(APPLICATION_ACTIVITY_TYPE)
  }
3. <- ASS.startActivityUncheckedLocked() { line:2116
    launchFlags & (Intent.FLAG_ACTIVITY_NEW_TASK|Intent.FLAG_ACTIVITY_MULTIPLE_TASK) != 0 或 launchSingleInstance 或 launchSingleTask:
      if(没有要求lauch到某个task && 没有要result reply) && 找到已有的task
        && 当前 (ActivityRecord curTop) 非空 && curTop.task != (ActivityRecord 要启动的 intentActivity).task
        && (找不到启动来源(sourceRecord == null) || 启动来源是topActivity)
        && (launchFlags & (FLAG_ACTIVITY_NEW_TASK|FLAG_ACTIVITY_TASK_ON_HOME) != 0):
          movedHome = true
          intentActivity.task.setTaskToReturnTo(HOME_ACTIVITY_TYPE)
    总结: TASK_ON_HOME启动时 设置返回到 HOME
}
4. <- ASS.startActivityUncheckedLocked() { line:2334
    if(!movedHome) && launchFlags & (NEW_TASK|TASK_ON_HOME) != 0:
      r.task.setTaskToReturnTo(HOME_ACTIVITY_TYPE)
} 总结: TASK_ON_HOME启动时 设置返回到 HOME
5. <- ASS.startActivityUncheckedLocked() { line:2452
    if 启动来源 是 最近应用:
      r.task.setTaskToReturnTo(RECENTS_ACTIVITY_TYPE)
} 总结: 启动来源 是 最近应用 时 设置返回到 最近应用
6. <- ASS.findTaskToMoveToFrontLocked() {
    if flags & ActivityManager.MOVE_TASK_WITH_HOME:
      task.setTaskToReturnTo(HOME_ACTIVITY_TYPE)
}
7. <- ActivityStack.resumeTopActivityInnerLocked(prev, bundle) { line:1637
    nextTask = topRunningActivityLocked().task
    if prevTask != nextTask && prevTask != topTask():
      taskNdx = mTaskHistory.indexOf(prevTask) + 1
      mTaskHistory.get(taskNdx).setTaskToReturnTo(HOME_ACTIVITY_TYPE)
} 总结: resumeTopActivity 时 prevTask != topTask 设置prevTask顶上那个task 返回到 HOME
8. <- AS.insertTaskAtTop(task, newActivity) { line:2033
    if task.isOverHomeStack() && nextTask != null:
      nextTask.setTaskToReturnTo(task.getTaskToReturnTo())
} 总结: 要插入到top的task 原来的nextTask 也返回到 task.getTaskToReturnTo()
9. <- AS.insertTaskAtTop(task, newActivity) { line:2043
    if isOnHomeDisplay():
      fromHome = lastStack.isHomeStack() 上次焦点stack
      if !isHomeStack() && (fromHome || topTask() != task):
        task.setTaskToReturnTo(fromHome? lastStack.topTask() == null? HOME_ACTIVITY_TYPE : lastStack.topTask().taskType : APPLICATION_ACTIVITY_TYPE)
    else:
      task.setTaskToReturnTo(APPLICATION_ACTIVITY_TYPE)
}
10. <- AS.moveTaskToBackLocked(taskId) { line:3751
    canGoHome = !tr.isHomeStack() && tr.isOverHomeStack()
    if canGoHome:
      nextTask.setTaskToReturnTo(tr.getTaskToReturnTo())
} 总结: if canGoHome 设置 要移动的task 原来的nextTask 返回到 task.getTaskToReturnTo()
11. <- AS.moveTaskToBackLocked(taskId) { line:3770
    遍历 mTaskHistory, 确保至少有一个task(自身)可以返回到home
    if(taskNdx == 1): task.setTaskToReturnTo(HOME_ACTIVITY_TYPE)
}
12. <- AS.moveTaskToBackLocked(taskId) { line:3788
    if(prevIsHome || (task == tr && canGoHome) || (numTasks <= 1 && isOnHomeDisplay())):
      taskToReturnTo = tr.getTaskToReturnTo()
      tr.setTaskToReturnTo(APPLICATION_ACTIVITY_TYPE)
      return mStackSupervisor.resumeHomeStack(taskToReturnTo, null, "moveTaskToBack")
}
13. <- AS.removeTask(task, reason, notMoving) { line:4306
    if task.isOverHomeStack() && taskNdx < topTask && !nextTask.isOverHomeStack():
        nextTask.setTaskToRetrunTo(HOME_ACTIVITY_TYPE)
}
```

# BluetoothControllerImpl

## 问题

updateInfo() 调用线程不一致问题 导致的 NPE

## 分析

```
BCI.handleConnectionChange()
1. <-- BCI.handleUpdateConnectionStates()
  1.1 <- (BCI.H BCI.mHandler) MSG_UPDATE_CONNECTION_STATES
  1.2 <- (BCI.H BCI.mHandler) MSG_ADD_PROFILE
  1.3 <- (BCI.H BCI.mHandler) MSG_REM_PROFILE
2. <--handler-- (BCI.H BCI.mHandler) MSG_UPDATE_SINGLE_CONNECTION_STATE 构造函数传入的bgLooper, 即 PhoneStatusBar 传入的 mHandlerThread.getLooper()

BCI.updateInfo() 保证 ArrayMap mDeviceInfo 键值对不为null
1. <- BCI.handleUpdateBondedDevices()
  <- (BCI.H BCI.mHandler) MSG_UPDATE_BONDED_DEVICES
2. <- BCI.handleUpdateConnectionStates()
3. <- BCI.handleUpdateConnectionState()
  3.1 <- BCI.handleUpdateConnectionStates()
  3.2 <- (BCI.H BCI.mHandler) MSG_UPDATE_SINGLE_CONNECTION_STATE
4. <- BCI.Receiver.onReceive() 跑在 receiver 回调线程. BCI.Receiver.ctor() 中 mContext.registerReceiver(this, filter) 指定了在ui主线程
```

## 解决

mContext.registerReceiver(this, filter, null, mHandler) 指定在handler(bgLooper)线程