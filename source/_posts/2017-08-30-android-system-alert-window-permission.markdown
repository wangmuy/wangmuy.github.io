---
layout: post
title: "android-system-alert-window-permission"
date: 2017-08-30 14:56:59 +0800
comments: true
categories: 
keywords: 
description: 
published: false
---

# Alert Window 弹窗控制

  # aosp-5.1.1_r6

  * 源码分析
```
WMS.addWindow()
-> PhoneWindowManager.checkAddPermission() {
  switch TYPE_SYSTEM_ALERT/TYPE_SYSTEM_OVERLAY/TYPE_SYSTEM_ERROR/TYPE_PHONE/TYPE_PRIORITY_PHONE:
    permission = android.Manifest.permission.SYSTEM_ALERT_WINDOW
    outAppOp[0] = AppOpsManager.OP_SYSTEM_ALERT_WINDOW (int数值24)
  if mContext.checkCallingOrSelfPermission(permission) != PackageManager.PERMISSION_GRANTED:
    return WindowManagerGlobal.ADD_PERMISSION_DENIED
}
-> ContextImpl.checkCallingOrSelfPermission(String permission)
-> ContextImpl.checkPermission(permission, Binder.getCallingPid(), Binder.getCallingUid())
-binder-> AMS.checkPermission(permission, pid, uid)
-> AMS.checkComponentPermission(permission, pid, UserHandle.getAppId(uid), owningUid=-1, exported=true)
-> ActivityManager.checkComponentPermission(permission, uid, owningUid, exported) {
  if uid==0 || uid == Process.SYSTEM_UID: 允许
  if UserHandle.isIsolated(uid): 禁止
  if owningUid >= 0 && UserHandle.isSameApp(uid, owningUid): 允许
  if !exported: 禁止
  if permission == null: 允许
  return AppGlobals.getPackageManager().checkUidPermission(permission, uid)
}

```

  * 解决方法

```java
if (permission == android.Manifest.permission.SYSTEM_ALERT_WINDOW) {
    final int callingUid = Binder.getCallingUid();
    if (callingUid == Process.SYSTEM_UID) {
        return WindowManagerGlobal.ADD_OKAY;
    }
    try {
        // 方法1: 查相同uid的有无visible的进程
        List<RunningAppProcessInfo> processInfos = ActivityManagerNative.getDefault().getRunningAppProcesses();
        for(RunningAppProcessInfo p: processInfos) {
            if (p.uid == callingUid) {
                Log.d(TAG, "wzd checkAddPermission callingUid=p " + p);
                if (p.importance <= RunningAppProcessInfo.IMPORTANCE_VISIBLE) {
                    Log.d(TAG, "wzd checkAddPermission p.importance=" + p.importance);
                }
            }
        }
        // 方法2: 查 top uid 是否与调用者相同
        List<RunningTaskInfo> tasks = ActivityManagerNative.getDefault().getTasks(1, 0);
        if (tasks.size() > 0 && tasks.get(0).baseActivity != null) {
            final String topPkgName = tasks.get(0).baseActivity.getPackageName();
            final int topUid = mContext.getPackageManager().getPackageUid(topPkgName, 0);
            Log.d(TAG, "checkAddPermission callingUid=" + callingUid + ", topPkgName=" + topPkgName + ", topUid=" + topUid);
            if (topUid == callingUid) {
                return WindowManagerGlobal.ADD_OKAY;
            }
        }
    } catch(RemoteException e) {
        Log.d(TAG, "checkAddPermission: " + e);
        // ignored
    } catch(NameNotFoundException e) {
        Log.d(TAG, "checkAddPermission: " + e);
        // ignored
    } catch(Exception e) {
        Log.d(TAG, "checkAddPermission: " + e);
        // ignored
    }
}
```

  # aosp-6.0.0_r1

  * 源码分析

```
WMS.addWindow()
-> PhoneWindowManager.checkAddPermission() {
    switch TYPE_SYSTEM_ALERT/TYPE_SYSTEM_OVERLAY/TYPE_SYSTEM_ERROR/TYPE_PHONE/TYPE_PRIORITY_PHONE:
      permission = android.Manifest.permission.SYSTEM_ALERT_WINDOW
      outAppOp[0] = AppOpsManager.OP_SYSTEM_ALERT_WINDOW (int数值24)
      if permission == android.Manifest.permission.SYSTEM_ALERT_WINDOW:
        mode = mAppOpsManager.checkOp(outAppOp[0], callingUid, attrs.packageName)
        switch mode == MODE_ALLOWED/MODE_IGNORED 返回 ADD_OKAY. 其中 MODE_IGNORED 会让添加的window在WMS中隐藏
          mode == MODE_ERRORED 返回 ADD_PERMISSION_DENIED
          default: mContext.checkCallingPermission(permission) == PERMISSION_GRANTED
}

1. AppOpsManager.checkOp(op=outAppOp[0], uid=callingUid, packageName=attrs.packageName)
-> AppOpsService.checkOperation(code=op, uid, packageName) {
    if isOpRestricted(uid, code, packageName): return MODE_IGNORED
    code = AppOpsManager.opToSwitch(code)
    UidState uidState = getUidStateLocked(uid, false)
    if uidState != null && uidState.opModes != null:
      uidMode = uidState.opModes.get(code)
      if uidMode 有设置不是ALLOWED: return uidMode
    Op op = getOpLocked(code, uid, packageName, false)
    if op == null: return AppOpsManager.opToDefaultMode(code)
    else: return op.mode
}
1.1 AppOpsService.isOpRestricted(uid, code, packageName) {
    userHandle = UserHandle.getUserId(uid) 多用户时 uid/100000, 否则 0
    opRestrictions = mOpRestrictions.get(userHandle)
    if opRestrictions != null && opRestrictions[code]:
      if AppOpsManager.opAllowSystemBypassRestriction(code):
        ops = getOpsLocked(uid, packageName, true)
        if ops != null && ops.isPrivileged: return false
      return true
    return false
}


2. ContextImpl.checkCallingPermission(String permission)
-> checkPermission(permission, pid=Binder.getCallingPid(), Binder.getCallingUid())
--binder--> AMS.checkPermission(permission, pid, uid)
-> ActivityManager.checkComponentPermission(permission, pid, uid, owningUid=-1, exported=true) {
    appid = UserHandle.getAppId(uid) uid 取模 100000
    PackageManager.checkUidPermission(permission, uid)
}
--binder--> PMS.checkUidPermission(permission, uid) {
    userId = UserHandle.getUserId(uid)
    obj = mSettings.getUserIdLPr(UserHandle.getAppId(uid))
    if(obj != null):
      PermissionState permissionState = ((SettingBase)obj).getPermissionState()
      if permissionState.hasPermission(permName, userId): return PERMISSION_GRANTED
    else:
      if mSystemPermissions.get(uid).contains(permName): return PERMISSION_GRANTED
}
```

  * 解决方法

```
AppOpsManager.checkOperation() 改为 AppOpsManager.checkOperationInner(), 新 AppOpsManager.checkOperation() 在末尾再加判断
```