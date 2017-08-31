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
WMS.addWindow() {
  if win.mAppOp != AppOpsManager.OP_NONE:
    if mAppOps.startOpNoThrow(win.mAppOp, win.getOwningUid(), win.getOwningPackage() != AppOpsManager.MODE_ALLOWED)
      (WindowState win).setAppOpVisibilityLw(false) # WindowState.hideLw() 隐藏窗口
}

AppOpsManager.startOpNoThrow(op, uid, packageName)
--binder--> AppOpsService.startOperation(token=getToken(mService), code=op, uid, packageName)
```

  * 解决方法

```
AppOpsManager.startOpNoThrow() 改为 AppOpsManager.startOpNoThrowInner(), 新 AppOpsManager.startOpNoThrow() 在末尾再加判断
```

  # aosp-6.0.0_r1

  * 源码分析

```
WMS.addWindow()
-> PhoneWindowManager.checkAddPermission() {
    switch TYPE_SYSTEM_ALERT/TYPE_SYSTEM_OVERLAY/TYPE_SYSTEM_ERROR/TYPE_PHONE/TYPE_PRIORITY_PHONE:
      permission = android.Manifest.permission.SYSTEM_ALERT_WINDOW
      outAppOp[0] = AppOpsManager.OP_SYSTEM_ALERT_WINDOW (int数值24)
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