---
layout: post
title: "ml-torch-windows-build"
date: 2017-04-16 22:31:21 +0800
comments: true
categories: 
keywords: 
description: 
#published: false
---

# Prereq
* VS2015 community
* conda 32bit ( Anaconda / Miniconda )
* conda 目录和其 Scripts 子目录加入 path


# 下载

```bash
git clone https://github.com/BTNC/distro-win.git
cd distro-win git checkout 2f95bf3ac5c6653b12ce7e5db1c8c0564c501b99 # 此次编译成功的版本
git submodule update
```


# 修改

install-deps.bat 可以注释掉开头对平台的判断, 直接写上要编译的平台

```bat
REM if     "%TORCH_VS_PLATFORM%" == "x86"                       set TORCH_VS_TARGET=x86
REM if not "%TORCH_VS_PLATFORM%" == "%TORCH_VS_PLATFORM:_x86=%" set TORCH_VS_TARGET=x86
REM if not "%TORCH_VS_PLATFORM%" == "%TORCH_VS_PLATFORM:_arm=%" set TORCH_VS_TARGET=arm
REM if     "%TORCH_VS_TARGET%"   == ""                          set TORCH_VS_TARGET=x64
set TORCH_VS_PLATFORM=x86
set TORCH_VS_TARGET=x86
```

```
th\extra\nn\lib\THNN\generic\TemporalRowConvolution.c:126 行
memcpy 改为 memmove, 不改编译nn会报错 fatal error C1001
```


# 编译

```
install.bat
```


# 测试

## 输入行太长 错误
th\install\luajit.cmd 按以下注释, 解决 path设置导致的 输入行太长 错误

```
REM call %TORCH_INSTALL_DIR%\torch-activate.cmd
```

## 运行测试

```
install\torch-activate.cmd
test.bat
```