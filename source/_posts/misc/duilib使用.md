---
layout: post
title: "duilib使用小结"
date: 2016-07-07 08:42:42 +0800
categories: [misc]
tags: 
description: 
---

版本: https://github.com/duilib/duilib master

环境: vs2013

# 基本内容
新建空的win32工程

```c++
#include <UILib.h>
class CDuiFrameWnd : public WindowImplBase
{
public:
  virtual LPCTSTR GetWindowClassName() const { return _T("DUIMainFrame"); }
  virtual CDuiString GetSkinFile() { return _T("duilib.xml"); }
  virtual CDuiString GetSkinFolder() { return _T(""); }
};

int APIENTRY _tWinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPTSTR lpCmdLine, int nCmdShow)
{
    CPaintManagerUI::SetInstance(hInstance);

    CDuiFrameWnd duiFrame;
    duiFrame.Create(NULL, _T("DUIWnd"), UI_WNDSTYLE_FRAME, WS_EX_WINDOWEDGE);
    duiFrame.CenterWindow();
    duiFrame.ShowModal();
    return 0;
}
```

# xml布局
duilib.xml 需要同步到exe目录

```xml
<Button name="closebtn" tooltip="关闭"   float="true" pos="44,5,74,24" width="28" normalimage=" file='SysBtn\CloseNormal.bmp' " hotimage=" file='SysBtn\CloseFocus.bmp' " pushedimage=" file='SysBtn\CloseFocus.bmp' "/>
```
tooltip="关闭"         提示文字
float="true"           绝对定位，其位置由pos属性指定

布局可以 VerticalLayout 为根节点, HorizontalLayout/VerticalLayout 穿插搭配. 某个Layout指定宽高, 可以用同类型做占位填充剩余空间:

```xml
<VerticalLayout>
  <!-- 占位 -->
  <HorizontalLayout />
  <!-- 底部 -->
  <HorizontalLayout height="30">
  </HorizontalLayout>
</VerticalLayout>
```

# 初始化
重载 InitWindow:

```c++
virtual void WindowImplBase::InitWindow()
{
  CEditUI* pEdit = static_cast<CEditUI*>(m_PaintManager.FindControl(_T("startMac")));
}
```

# 自绘控件
TODO

# 事件消息处理 [^1]
方法1: 重载 Notify

```c++
virtual void WindowImplBase::Notify(TNotifyUI& msg)
{
    if( msg.sType == _T("windowinit") )
    {
    }      
    else if( msg.sType == DUI_MSGTYPE_CLICK ) // 对应 _T("click")
    {
      if (msg.pSender->GetName() == _T("btnGen")) {}
    }
    else if (msg.pSender->GetName() == _T("btnClose"))
	{
		PostMessage(WM_CLOSE); // 退出
	}
}
```

# 资源
增加rc资源文件

## xml和图片资源(假设全在根目录下, 不包括子目录)

### raw
直接放在根目录

### zip
xml 和所有资源 打包成zip, 放在跟目录

```c++
CPaintManagerUI::SetResourcePath(CPaintManagerUI::GetInstancePath());
CPaintManagerUI::SetResourceZip(_T("res.zip"));
```

### rc资源raw
TODO

### rc资源zip [^2]
rc资源 添加 导入zip, 类型为 ZIPRES

```c++
virtual UILIB_RESOURCETYPE WindowImplBase::GetResourceType() const
{ return UILIB_ZIPRESOURCE; }
virtual LPCTSTR WindowImplBase::GetResourceID() const
{ return MAKEINTRESOURCE(IDR_ZIPRES1); }
```

## 静态库
重编编译 DuiLib项目: 选择对应的 debug/release 版本, 设置为编译静态库, 添加 UILIB_STATIC 预编译宏. 编译出的lib(release版)改名 DuiLib_s.lib(unicode版名 DuiLib_us.lib), 放入Lib目录

```c++
#define UILIB_STATIC
#include <UIlib.h>
```

```c++
#ifdef UILIB_STATIC
#ifdef _DEBUG
#  ifdef _UNICODE
#    pragma comment(lib, "DuiLib_uds.lib")
#  else
#    pragma comment(lib, "Duilib_ds.lib")
#  endif
#else
#  ifdef _UNICODE
#    pragma comment(lib, "DuiLib_us.lib")
#  else
#    pragma comment(lib, "DuiLib_s.lib")
#  endif
#endif
#else
#ifdef _DEBUG
#  ifdef _UNICODE
#    pragma comment(lib, "DuiLib_ud.lib")
#  else
#    pragma comment(lib, "Duilib_d.lib")
#  endif
#else
#  ifdef _UNICODE
#    pragma comment(lib, "DuiLib_u.lib")
#  else
#    pragma comment(lib, "DuiLib.lib")
#  endif
#endif
#endif
```

## 图标 [^3]
exe静态图标:
添加 资源 -> Icon 导入

exe运行图标:
duiFrame.create() 之后调用

```c++
duiFrame.SetIcon(IDI_ICON);
```

## 添加版本信息
添加 资源 -> Version

# 其他 [^4]

## 设置标题栏区域
xml Window节点 caption属性, 指定标题栏区域(top,left,right,bottom)

```xml
<Window size="800,600" caption="0,0,0,32">
```

## 窗口大小调整
xml Window节点 sizebox属性, 边缘可调范围

```xml
<Window size="800,600" sizebox="4,4,4,4">
```

## 窗口最小尺寸
xml Window节点 mininfo属性

```xml
<Window size="800,600" mininfo="600,400">
```

_______________________________________________________________________________
[^1]: [duilib-事件处理](http://www.cnblogs.com/Alberl/p/3352904.html)
[^2]: [duilib-zip-rc](http://blog.csdn.net/x356982611/article/details/39271205)
[^3]: [win32-icon](http://stackoverflow.com/questions/320677/how-do-i-set-the-icon-for-my-application-in-visual-studio-2008)
[^4]: [duilib-misc](http://www.cnblogs.com/Alberl/p/3354294.html)
