---
layout: post
title: "PackageInstaller-side-of-dynamic-permissions"
date: 2016-11-29 10:57:43 +0800
comments: true
categories: 
keywords: 
description: 
#published: false
---

# manifest统计

.PackageInstallerActivity
action="android.content.pm.action.CONFIRM_PERMISSIONS"
category="android.intent.category.DEFAULT"

.permission.ui.GrantPermissionsActivity
action="android.content.pm.action.REQUEST_PERMISSIONS"
category="android.intent.category.DEFAULT"

.permission.ui.ManagePermissionsActivity
action="android.intent.action.MANAGE_PERMISSIONS"
action="android.intent.action.MANAGE_APP_PERMISSIONS"
action="android.intent.action.MANAGE_PERMISSION_APPS"
category="android.intent.category.DEFAULT"

receiver=".permission.model.PermissionStatusReceiver"
android:permission="android.permission.GRANT_RUNTIME_PERMISSIONS"
intent-filter action="android.intent.action.GET_PERMISSIONS_COUNT"

# PackageInstallerActivity
入口activity
CONFIRM_PERMISSIONS 通过session进来, 记住sessionId, 选择后回调给session mInstaller.setPermissionsResult()

# GrantPermissionsActivity
应用主动申请权限时出现的授权界面
授权权限按组进行

1. 初始化 updateDefaultResults() deny all
2. devicePolicyManager.getPermissionPolicy() 获取默认permission类型(STATE_DEFAULT运行时/AUTO_DENY/AUTO_GRANT)
根据 permissionPlicy 自动授权 AUTO_DENY/AUTO_GRANT 给 non-fixed( !group.isUserFixed() && !group.isPolicyFixed())类型的权限
3. 剩下的STATE_DEFAULT类型更新界面让用户授权

# ManagePermissionsActivity
action="android.intent.action.MANAGE_PERMISSIONS"
ManagePermissionsFragment

action="android.intent.action.MANAGE_APP_PERMISSIONS"
AppPermissionsFragment

action="android.intent.action.MANAGE_PERMISSION_APPS"
PermissionAppsFragment

# PermissionStatusReceiver
当有 Intent.ACTION_GET_PERMISSIONS_COUNT 广播询问时, 获取对应的package授权权限数量, 广播反馈
/packages/apps/Settings 用到