---
layout: post
title: "aosp-build-makefiles-anatomy"
date: 2016-07-02 09:10:10 +0800
comments: true
categories: 
keywords: 
description: 
published: false
---

aosp 5.1.1_r6

# build/envsetup.sh
===============================================================

找device和vendor目录下的vendorsetup.sh 并 source
addcompletions() 找 sdk/bash_completion 下 [a-z]*.bash 并 source
  function gettop{} 返回aosp编译根目录
  
  function lunch{}
    TARGET_PRODUCT=$product
    TARGET_BUILD_VARIANT=$variant
    TARGET_BUILD_TYPE=release
    aosp_strawberry-userdebug 其中 product==aosp_strawberry, variant==userdebug

设置shell变量, 如 $ANDROID_BUILD_TOP


# 包含关系
===============================================================
$ANDROID_BUILD_TOP/Makefile
-> build/core/main.mk 定义 TOPDIR 为空, BUILD_SYSTEM := $(TOPDIR)build/core
  -> $(BUILD_SYSTEM)/help.mk
  -> $(BUILD_SYSTEM)/config.mk( 定义 BUILD_XXX 对应的各个mk, 供以后include )
    -> $(BUILD_SYSTEM)/envsetup.mk
      -> $(BUILD_SYSTEM)/product_config.mk
        -> $(BUILD_SYSTEM)/node_fns.mk
        -> $(BUILD_SYSTEM)/product.mk
        -> $(BUILD_SYSTEM)/device.mk
        -> $(TARGET_PRODUCT)(来自envsetup.sh) 的 AndroidProducts.mk {
          call product.mk:import-products
            -> node_fns.mk:import-nodes(间接定义 PRODUCTS==目标mk文件列表, 同时PRODUCTS本身也用作一个前缀)
            -> _import-nodes-inner -> import-node -> include
          定义 INTERNAL_PRODUCT== $(TARGET_PRODUCT)扩展出来的目标mk文件
          }
  -> $(BUILD_SYSTEM)/cleanbuild.mk
  -> $(BUILD_SYSTEM)/definitions.mk
  -> $(BUILD_SYSTEM)/dex_preopt.mk
  -> build/core/pdk_config.mk


# apicheck
===============================================================
apicheck.mk { .PHONY checkapi; droidcore: checkapi }
<- 

# Add-on
===============================================================
主mk: device/sample/products/sample_addon.mk
<- device/sample/products/AndroidProducts.mk
<- build/core/product.mk
  function  _find-android-products-files
  查找 device, vendor, $(SRC_TARGET_DIR)/product 目录下所有 AndroidProducts.mk
  <- function get-all-product-makefiles
     返回排序好的所有 AndroidProducts.mk 定义的 PRODUCT_MAKEFILES
    <- build/core/product_config.mk
       all_product_configs 从中找到对应 product 的 makefile
       all_product_makefiles 集合所有product的makefile
       call import-products 导入product的makefile

例:
device/sample/products/AndroidProducts.mk
PRODUCT_MAKEFILES := $(LOCAL_DIR)/sample_addon.mk
PRODUCT_PACKAGES := PlatformLibraryClient \
	com.example.android.platform_library \
	libplatform_library_jni

依赖 com.example.android.platform_library 来自
device/sample/frameworks/PlatformLibrary/Android.mk
<- device/sample/frameworks/Android.mk 引入所有子目录mk
<- device/sample/Android.mk 引入所有子目录mk

使用到的主mk: build/core/tasks/sdk-addon.mk
1. MODULES 由 fils_to_copy 拷贝到 dest
2. COPY_FILES 由 files_to_copy 拷贝到 dest
3. TARGET_CPU_API对应的img 拷贝到 $(addon_dir_img)
4. 修改 TARGET_CPU_API对应的 source.properties, 拷贝到 $(addon_dir_img)
5. doc_module 拷贝到 $(OUT_DOCS)
6. sdk打包, image打包


function module-installed-files 查找某个module在out目录安装位置

# Overlay
===============================================================

# Jack

java-to-jack(definitions.mk)
<- LOCAL_IS_STATIC_JAVA_LIBRARY 构建static javalib 时候 $(full_classes_jack)的命令 (static_java_library.mk)
