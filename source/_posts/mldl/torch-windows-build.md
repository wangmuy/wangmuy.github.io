---
layout: post
title: "Torch windows 编译"
date: 2017-04-16 22:31:21 +0800
categories: [mldl]
description: 
---

# Prereq
* VS2015 community
* conda ( Anaconda / Miniconda )
* conda 目录和其 Scripts 子目录加入 path


# 下载

```bash
git clone https://github.com/BTNC/distro-win.git
cd distro-win; git checkout 2f95bf3ac5c6653b12ce7e5db1c8c0564c501b99 # 此次编译成功的版本
git submodule update
```


# 修改

* install-deps.bat 可以注释掉开头对平台的判断, 直接写上要编译的平台, conda env 名称增加平台类型

```bat
REM if     "%TORCH_VS_PLATFORM%" == "x86"                       set TORCH_VS_TARGET=x86
REM if not "%TORCH_VS_PLATFORM%" == "%TORCH_VS_PLATFORM:_x86=%" set TORCH_VS_TARGET=x86
REM if not "%TORCH_VS_PLATFORM%" == "%TORCH_VS_PLATFORM:_arm=%" set TORCH_VS_TARGET=arm
REM if     "%TORCH_VS_TARGET%"   == ""                          set TORCH_VS_TARGET=x64
REM 直接设置 x86 或 x64
set TORCH_VS_PLATFORM=x86
set TORCH_VS_TARGET=x86

...
REM if "%TORCH_INSTALL_DIR%" == "" set TORCH_INSTALL_DIR=%TORCH_DISTRO%\install
set TORCH_INSTALL_DIR=%TORCH_DISTRO%\install
...

:CONDA_SETUP

if "%TORCH_CONDA_ENV%" == "" set TORCH_CONDA_ENV=torch-vc%CONDA_VS_VERSION%-%TORCH_VS_TARGET%
```

* pkg\torch\lib\TH 无法解析的外部符号 *_AVX, *_AVX2

pkg\torch\lib\TH\THVector.c 修改

```c
#if defined(USE_AVX)
/*#include "vector/AVX.h"*/
#define __AVX__
#include "vector/AVX.c"
#endif

#if defined(USE_AVX2)
/*#include "vector/AVX2.h"*/
#define __AVX2__
#include "vector/AVX2.c"
#endif
```

* nn 编译报错 fatal error C1001

extra\nn\lib\THNN\generic\TemporalRowConvolution.c:126 行

```
memcpy 改为 memmove
```

* 其他错误

多数 submodule lua pkg 编译/安装错误 可以删除对应 submodule, 重新运行 install-deps.bat

# 编译

```
install.bat
```


# 测试

## 输入行太长 错误
install\luajit.cmd 按以下注释, 解决 path设置导致的 输入行太长 错误

```
REM call %TORCH_INSTALL_DIR%\torch-activate.cmd
```

## 运行测试

```
install\torch-activate.cmd
test.bat
```