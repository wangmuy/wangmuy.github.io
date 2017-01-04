---
layout: post
title: "activity-convertFromTranslucent"
date: 2016-12-01 10:16:21 +0800
comments: true
categories: 
keywords: 
description: 
#published: false
---

Activity.convertFromTranslucent() {
ActivityManagerNative.getDefault().convertFromTranslucent(mToken)
WindowManagerGlobal.getInstance().changeCanvasOpacity(mToken, true)
}

--binder--> ActivityManagerService.convertFromTranslucent() {
(ActivityRecord r).changeWindowTranslucency(true)
r.task.stack.releaseBackgroundResources(r)
mStackSupervisor.ensureActivitiesVisibleLocked(null, 0)
mWindowManager.setAppFullscreen(token, true)
}
-> ActivityStackSupervisor.ensureActivitiesVisibleLocked() {
遍历 display, stack
  ActivityStack.ensureActivitiesVisibleLocked() {
    behindFullscreen && r.visible: 影响到的(ActivityRecord r) 根据r.state动作:
    case RESUMED: mStackSupervisor.mStoppingActivities.add(r); mStackSupervisor.scheduleIdleLocked()
  }
}

--IDLE_NOW_MSG--> ActivityStackSupervisor.activityIdleInternelLocked() {
ArrayList<ActivityRecord> stops = processStoppingActivitiesLocked(true) 从mStoppingActivities取要stop的
遍历stops: (ActivityStack stack).stopActivityLocked(r)
}

-> ActivityStack.stopActivityLocked(r) { r.app.thread.scheduleStopActivity() }
-> ApplicationThreadProxy.scheduleStopActivity(r.appToken, r.visible==false, r.configChangeFlags)
--binder--> (ActivityThread.ApplicationThread extends ApplicationThreadNative).scheduleStopActivity() { scheduleStopActivity() }
--handler STOP_ACTIVITY_HIDE--> ActivityThread.handleStopActivity()
-> ActivityThread.performStopActivityInner()
-> Activity.performStop()
-> Instrumentation.callActivityOnStop() { activity.onStop() }
