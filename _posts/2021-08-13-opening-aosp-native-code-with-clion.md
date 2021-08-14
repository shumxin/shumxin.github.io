---
layout: post
title:  "使用 CLion 查看 AOSP Native 代码"
date:   2021-08-13 23:56:40 +0800
categories: android
---

## 一. 引言

`Android Studio` 对于 `AOSP Native` 部分的代码支持还不够完善，比如不支持跳转，无法查看引用以及无法通过 `CTRL + F12` 查看代码结构，给代码阅读造成了极大的不便。查看了谷歌[官方文档](https://android.googlesource.com/platform/build/soong/+/refs/heads/master/docs/clion.md)，了解到可以通过配置 `CMakeLists.txt` 导入到 `CLion` 来进行阅读 `Native` 部分代码。

## 二. 源码环境

本人使用的是 [`ProtonAOSP`](https://github.com/ProtonAOSP/android_manifest)，基于 `AOSP` 的三方 `ROM` 开源项目

## 三. 生成 `CLion` 项目

### 1. 配置相关环境变量用于生成 `CMakeLists.txt`

```shell
$ export SOONG_GEN_CMAKEFILES=1
$ export SOONG_GEN_CMAKEFILES_DEBUG=1
```

### 2. 编译相关模块

```shell
$ source build/envsetup.sh
$ mmm frameworks/av/media/mediaserver
```

```shell
============================================
PLATFORM_VERSION_CODENAME=REL
PLATFORM_VERSION=11
TARGET_PRODUCT=aosp_arm
TARGET_BUILD_VARIANT=eng
TARGET_BUILD_TYPE=release
TARGET_ARCH=arm
TARGET_ARCH_VARIANT=armv7-a-neon
TARGET_CPU_VARIANT=generic
......
============================================
......
[100% 4951/4951] Install: out/target/product/generic/system/bin/mediaserver
```

### 3. 导入 `CLion`

```shell
$ ls -l out/development/ide/clion/frameworks/av/media/mediaserver/mediaserver-arm-android
total 44
drwxrwxr-x 4 shumxin shumxin  4096 8月  13 23:35 cmake-build-mediaserver
-rw-rw-r-- 1 shumxin shumxin 40090 8月  13 23:28 CMakeLists.txt
```

### 3. 在一个项目里面整合多个源码目录

比如想看 `mediaserver` 相关的 `Native` 代码，其有涉及到其他模块，可以重新创建一个 `CMakeList.txt` 放在 `out/development/ide/clion/frameworks` 下，将相关模块整合进来，如下：

```cmake
cmake_minimum_required(VERSION 3.6)
project(frameworks)
add_subdirectory(av/media/libmediaplayerservice/libmediaplayerservice-arm-android)
add_subdirectory(av/media/mediaserver/mediaserver-arm-android)
add_subdirectory(native/cmds/servicemanager/servicemanager-arm-android)
add_subdirectory(native/libs/binder/libbinder-arm-android)
```

![aosp_native_code_in_clion](/res/images/20210813_aosp_native_code_in_clion.png){:height="550px" width="720px"}