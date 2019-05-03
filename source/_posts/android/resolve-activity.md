---
layout: post
title: "ResolverActivity 流程"
date: 2016-11-16 15:12:28 +0800
categories: [android]
description: 
---

版本 aosp-5.1.1_r6
============================================================

```
ResolverActivity.java ResolveListAdapter.ctor()
-> ResolveListAdapter.rebuildList()
--binder--> PackageManagerService.queryIntentActivities() {
    if intent有component, 直接返回component指定的;
    if intent有packageName, ActivityIntentResolver.queryIntentForPackage()
    else ActivityIntentResolver.queryIntent()
}
->1. (ActivityIntentResolver父类 IntentResolver).queryIntent()
->2. (ActivityIntentResolver父类 IntentResolver).queryIntentForPackage()

-> IntentResolver.buildResolveList()
```

<!--more-->

```
PMS.findPreferredActivity()
1. <- PMS.setLastChosenActivity()
<- ResolverActivity.onIntentSelected()

2. <- PMS.getLastChosenActivity()
<- ResolverActivity.rebuildList()

3. <- PMS.chooseBestActivity() {findPreferredActivity(intent,resolvedType,flags,query,r0.priority,true==always,false==removeMatches,debug,userId)}
<- PMS.resolveIntent()

4. <- PMS.getHomeActivities() {
  list=queryIntentActivities(); return findPreferredActivity(list) 返回NULL表示没有preferred }
<--binder-- PM.getHomeActivities()
4.1 <- SystemUI SystemServicesProxy.getHomeActivityPackageName()
  <- SystemUI AlternateRecentsComponent.startRecentsActivity() 最近应用页面
4.2 <- ApplicationPackageManager.getHomeActivities() ContextImpl.mPackageManager==ApplicationPackageManager, 实际client端的PackageManager, api公开接口
4.3 <- PackageManagerBackupAgent.getPreferredHomeComponent()
  <- PMBA.onBackup()
```

<!--

定死Home应用:
初始方法:
加在PMS.resolveIntent() {
  query=queryIntentActivities();
  if(query含Launcher){滤掉其他或直接返回唯一};
  return chooseBestActivity(query) }
缺陷:
PMS.queryIntentActivities()没有走这个流程(Recents等有调用此接口)
解决:
queryIntentActivities() 有多个return出口, 不好直接改写
将 queryIntentActivities() 包起来, 改名 queryIntentActivityesInner(), 重新提供一个
queryIntentActivities() {
  query=queryIntentActivityesInner();
  if(query含Launcher){滤掉其他或直接返回唯一} }

-->
